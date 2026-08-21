# Arquitetura — FileBridge

## Visão geral

FileBridge é uma ferramenta de transferência **efêmera, ponto-a-ponto, baseada em sessão**. Não é um app de chat persistente, nem armazenamento em nuvem, nem plataforma de mensagens geral.

**Caso de uso de origem:** transferir arquivos e saída de terminal de uma VM Linux para o host, sem habilitar pasta compartilhada e sem estar na mesma rede.

A arquitetura também suporta **compartilhamento de tela/vídeo em tempo real** utilizando a mesma conexão WebRTC já estabelecida para arquivos e mensagens, sem criar uma segunda conexão P2P ou um servidor de mídia.

---

## Como funciona

```text
Dispositivo A                    Firebase (signaling)              Dispositivo B
     │                                   │                               │
     │──── gera código de sessão ───────►│                               │
     │                                   │◄──── digita o código ─────────│
     │◄──────── troca SDP/ICE (WebRTC handshake, ~1-2s) ─────────────────►│
     │                                   │                               │
     │══════════ conexão P2P direta estabelecida, Firebase sai de cena ══│
     │                                   │                               │
     │◄════════ arquivos e mensagens trafegam direto, criptografados ══►│
     │                                   │                               │
     │──── opcional: adiciona video track à conexão existente ──────────►│
     │◄══════════════ vídeo/tela via WebRTC, em paralelo ═══════════════►│
```

1. Dispositivo A gera um código de sessão curto e legível (ex: `TIGER-42`)
2. Dispositivo B digita o código
3. Os dois apps trocam um handshake WebRTC via Firebase Realtime Database (~1–2 segundos)
4. Firebase é descartado. Todo o restante flui **direto entre os dispositivos via WebRTC**
5. Arquivos e mensagens utilizam **WebRTC DataChannel**
6. Compartilhamento de tela/vídeo utiliza um **video track adicional na mesma `RTCPeerConnection`**
7. A sessão termina quando qualquer um dos dois dispositivos fecha a sessão/app

---

## Sessão e código de conexão

A sessão é efêmera e identificada por um código curto, legível e fácil de digitar.

Exemplo:

```text
TIGER-42
```

O código é apenas um identificador conveniente para localizar a sessão no serviço de sinalização. Ele não deve ser tratado como segredo criptográfico.

Para produção, o código deve possuir entropia suficiente para dificultar tentativas de descoberta de sessões ativas.

### Ciclo de vida

```text
sessão criada
      ↓
aguarda segundo dispositivo
      ↓
handshake WebRTC
      ↓
P2P estabelecido
      ↓
Firebase deixa de participar
      ↓
transferência / mensagens / vídeo
      ↓
sessão encerrada
      ↓
limpeza local
```

Sessões abandonadas devem possuir **TTL/expiração automática** no Firebase para impedir acúmulo de entradas temporárias.

---

## Comunicação P2P

A comunicação principal utiliza uma única `RTCPeerConnection`.

### DataChannel

Responsável por:

* mensagens de texto
* transferência de arquivos
* metadados de transferência
* progresso
* comandos de controle
* telemetria necessária para o compartilhamento de vídeo
* eventual renegociação de mídia após o handshake inicial

### Video Track

Responsável por:

* compartilhamento de tela
* posteriormente, outros tipos de vídeo, caso sejam adicionados

O video track é adicionado à **mesma `RTCPeerConnection`** usada pelo DataChannel.

```text
                    RTCPeerConnection
                           │
             ┌─────────────┴─────────────┐
             │                           │
        DataChannel                  Video Track
             │                           │
      ┌──────┼──────┐                tela/vídeo
      │      │      │
    texto  arquivo  controle
```

Não é criada uma segunda conexão WebRTC apenas para o vídeo.

---

## Buffer / histórico

Cada dispositivo mantém um **buffer circular local** (padrão: 20 itens).

Itens = mensagens de texto + anexos de arquivo, contados juntos.

No overflow:

1. O item mais antigo é removido da memória
2. Seu arquivo associado é apagado do disco
3. O próximo item passa a ocupar sua posição

Não existe banco de dados persistente para o histórico.

Tudo é mantido temporariamente:

* metadados em memória
* arquivos em diretório temporário/local

Ao fechar o app, a sessão e o buffer são descartados.

O tamanho do buffer será configurável em ajustes futuramente.

### Compartilhamento de vídeo

O vídeo **não entra no buffer**.

Streaming é tratado como mídia efêmera em tempo real e não deve ser armazenado como histórico de mensagens ou arquivos.

---

## Por que Firebase, por que não P2P puro

Dois dispositivos em redes diferentes podem não conseguir estabelecer conexão diretamente devido a NAT, CGNAT, firewalls e outras restrições de rede.

Por isso, um intermediário de **sinalização** é necessário para permitir que os peers descubram como estabelecer a conexão.

Firebase Realtime Database (plano gratuito Spark) é utilizado **somente durante a sinalização inicial**:

```text
A ──► Firebase ◄── B
        │
        │ SDP / ICE
        ↓
   conexão P2P
```

Após a conexão:

```text
A ◄════════════════════════════► B
             WebRTC
```

Firebase deixa de participar do fluxo de dados.

### Fallback TURN

A arquitetura deve prever suporte a **TURN como fallback**, pois nem todas as redes permitem uma conexão P2P direta através de STUN.

O fluxo passa a ser:

```text
Tentativa P2P direta
       │
       ├── sucesso ──► conexão direta
       │
       └── falha ────► TURN
                           │
                           └── relay WebRTC
```

TURN é especialmente relevante para compartilhamento de vídeo, pois uma falha de conectividade que é apenas incômoda durante uma transferência de arquivo pode tornar um streaming inviável.

A decisão de provedor/custo de TURN permanece separada da implementação inicial.

---

## Privacidade — o que o Firebase vê

| O Firebase vê                                                | O Firebase NÃO vê                     |
| ------------------------------------------------------------ | ------------------------------------- |
| Candidatos ICE e SDP da negociação                           | Conteúdo dos arquivos transferidos    |
| Que uma sessão com determinado código existe temporariamente | Conteúdo das mensagens                |
| Informações necessárias para sinalização                     | Conteúdo do vídeo/tela                |
| Dados necessários enquanto o handshake ocorre                | Dados transferidos após a conexão P2P |

A comunicação WebRTC utiliza os mecanismos de segurança do próprio WebRTC, incluindo DTLS para os canais de dados e SRTP para mídia.

> ⚠️ **As regras do Firebase deste repositório são apenas para desenvolvimento** (`.read: true / .write: true`). Antes de publicar em produção, restrinja as regras para permitir somente as operações necessárias para a criação/uso de sessões e implemente TTL/expiração.

---

# Compartilhamento de tela / vídeo

O compartilhamento de tela é uma extensão da sessão existente.

Ele **não abre uma nova conexão WebRTC nem introduz um servidor de mídia**.

Após o DataChannel estar estabelecido:

```text
DataChannel aberto
       ↓
usuário inicia compartilhamento
       ↓
captura da tela
       ↓
addTrack(videoTrack)
       ↓
renegociação
       ↓
vídeo recebido pelo peer
```

O video track trafega pela mesma `RTCPeerConnection`, em paralelo ao DataChannel.

```text
Dispositivo A                                                    Dispositivo B
     │                                                                  │
     │══ DataChannel já aberto (arquivos/mensagens) ═══════════════════►│
     │                                                                  │
     │── captura de tela ───────────────────────────────────────────────►│
     │                                                                  │
     │── renegociação SDP ─────────────────────────────────────────────►│
     │                                                                  │
     │══ video track via WebRTC, em paralelo ao DataChannel ═══════════►│
```

---

## Renegociação da sessão

O compartilhamento de vídeo pode exigir uma nova negociação SDP depois que a conexão inicial já estiver estabelecida.

A arquitetura deve priorizar a troca dessa sinalização **através do próprio DataChannel**, evitando retornar ao Firebase.

Fluxo esperado:

```text
PeerConnection estabelecida
         ↓
DataChannel aberto
         ↓
A adiciona video track
         ↓
A cria nova offer
         ↓
offer enviada pelo DataChannel
         ↓
B aplica offer
         ↓
B cria answer
         ↓
answer enviada pelo DataChannel
         ↓
A aplica answer
         ↓
vídeo começa
```

Essa abordagem mantém a propriedade de que o Firebase participa somente da criação da sessão.

A implementação deve validar essa renegociação em um protótipo antes da adoção definitiva em produção.

---

# Captura de tela

A captura de tela é específica de cada plataforma.

Não existe uma API cross-platform única capaz de oferecer exatamente o mesmo mecanismo em todas as plataformas suportadas.

### Android

Utilizar o mecanismo de captura do sistema baseado em `MediaProjection`.

### Windows / Linux / macOS

Utilizar os mecanismos de captura disponibilizados pelo `flutter_webrtc` e/ou integrações nativas apropriadas.

### iOS

A implementação é mais restrita e utiliza `ReplayKit`, incluindo a infraestrutura adicional exigida pelo sistema, como Broadcast Upload Extension.

A camada de captura deve ficar isolada de `media_negotiation` para evitar que código específico de plataforma se espalhe pelo restante da aplicação.

---

# Perfis de qualidade

O usuário poderá escolher entre três perfis fixos e um modo automático.

| Perfil         |         Resolução |               FPS | Comportamento     |
| -------------- | ----------------: | ----------------: | ----------------- |
| **Alta**       |         1920×1080 |            60 FPS | Máxima qualidade  |
| **Média**      |          1280×720 |            30 FPS | Equilíbrio        |
| **Baixa**      |           854×420 |            15 FPS | Menor consumo     |
| **Automático** | Perfil adaptativo | Perfil adaptativo | Ajuste automático |

> A resolução de `854×420` deve ser validada durante a implementação. Caso seja desejada maior compatibilidade com codificadores e players, `854×480` pode ser utilizado como alternativa.

A escolha manual de um perfil **não deve redimensionar dinamicamente a resolução**.

No modo automático, o principal mecanismo de adaptação será o **bitrate/qualidade de imagem**, mantendo a resolução e o FPS do perfil ativo sempre que o desempenho permitir.

---

# Qualidade automática

O modo automático funciona como um controlador adaptativo semelhante ao conceito de qualidade automática utilizado em serviços de streaming.

O objetivo é manter a melhor qualidade visual possível sem provocar travamentos persistentes.

### Exemplo

```text
720p30
   │
   ├── desempenho estável
   │        ↓
   │   aumenta qualidade/bitrate
   │
   └── desempenho degradando
            ↓
      reduz qualidade/bitrate
```

Caso a qualidade máxima daquele perfil ainda não seja suficiente:

```text
1080p60
   ↓
redução de qualidade/bitrate
   ↓
problema persiste
   ↓
troca para perfil inferior
   ↓
720p30
   ↓
problema persiste
   ↓
420p15
```

A subida de qualidade deve ser mais conservadora do que a redução.

### Histerese

Para evitar oscilações constantes:

```text
1080p60
   ↓ problema
720p30
   ↓
estabiliza
   ↓
não sobe imediatamente
   ↓
aguarda período de estabilidade
   ↓
tenta novamente aumentar qualidade
```

O sistema deve exigir um período mínimo de estabilidade antes de elevar novamente a qualidade.

---

# Adaptação baseada em telemetria

O `adaptive_quality` deve analisar estatísticas da conexão e do receptor.

Indicadores relevantes:

* FPS recebido
* frames perdidos
* packets lost
* jitter
* RTT
* bitrate efetivo
* estado da conexão
* capacidade de processamento observada
* sinais de sobrecarga do receptor

A decisão de qualidade não deve depender exclusivamente do uso de CPU do emissor.

### Feedback do receptor

O receptor pode informar ao emissor, através do DataChannel, métricas simplificadas de desempenho.

Exemplo:

```text
Receptor
   │
   │ fps=14
   │ dropped=18%
   │ render_pressure=high
   ▼
DataChannel
   │
   ▼
Emissor
   │
   └── reduz bitrate/qualidade
```

Isso é especialmente relevante em dispositivos antigos ou de baixo desempenho.

---

# Qualidade versus resolução

A adaptação automática deve priorizar **qualidade de compressão/bitrate** antes de modificar a resolução do perfil.

Exemplo:

```text
1920×1080 @ 60 FPS
        │
        ├── qualidade alta
        ├── qualidade média
        └── qualidade baixa
```

A resolução e o FPS continuam iguais dentro do perfil.

Somente quando o sistema concluir que aquele perfil não consegue mais ser mantido de maneira estável é que deve ocorrer uma mudança de perfil:

```text
1080p60
   ↓
720p30
   ↓
420p15
```

Dessa forma, o sistema evita que a imagem fique mudando de resolução constantemente durante uma sessão.

---

# Estrutura do código

```text
app/
└── lib/
    ├── core/
    │   ├── session/
    │   │   └── geração de código, ciclo de vida da sessão
    │   │
    │   ├── signaling/
    │   │   ├── firebase_signaling/
    │   │   │   └── handshake inicial, SDP e ICE
    │   │   └── renegotiation/
    │   │       └── renegociação de mídia via conexão existente
    │   │
    │   ├── transfer/
    │   │   ├── DataChannel
    │   │   ├── fragmentação de arquivos
    │   │   └── progresso
    │   │
    │   ├── buffer/
    │   │   ├── fila circular
    │   │   └── overflow → apaga arquivo + registro
    │   │
    │   └── media/
    │       ├── screen_share/
    │       │   ├── captura de tela
    │       │   └── controle do video track
    │       │
    │       ├── receiver/
    │       │   └── recebimento e renderização de vídeo
    │       │
    │       └── adaptive_quality/
    │           ├── quality_controller
    │           ├── quality_profile
    │           └── media_stats
    │
    ├── features/
    │   ├── home/
    │   │   └── tela: gerar código OU digitar código
    │   │
    │   ├── chat/
    │   │   └── tela: mensagens + anexos de arquivo
    │   │
    │   ├── screen_share/
    │   │   └── tela de configuração/visualização do stream
    │   │
    │   └── feedback/
    │       └── formulário de feedback no app
    │
    └── main.dart
```

---

# Stack técnica

| Camada                     | Tecnologia                                    | Justificativa                                                              |
| -------------------------- | --------------------------------------------- | -------------------------------------------------------------------------- |
| Framework do app           | Flutter (Dart)                                | Um código-fonte → Android, iOS, Windows, Linux, macOS                      |
| Transferência P2P          | `flutter_webrtc`                              | WebRTC DataChannel para arquivos/mensagens e WebRTC MediaStream para vídeo |
| Sinalização inicial        | Firebase Realtime Database                    | Sinalização efêmera para SDP/ICE                                           |
| Comunicação após handshake | WebRTC                                        | Comunicação direta entre peers sempre que possível                         |
| Fallback de conectividade  | TURN                                          | Permitir conexão em redes onde P2P direto falhar                           |
| Armazenamento local        | `path_provider` + fila em memória             | Sem banco persistente para o histórico da sessão                           |
| Gerenciamento de estado    | Riverpod                                      | Estado reativo e separação entre UI e lógica                               |
| Compartilhamento de tela   | `flutter_webrtc` + APIs nativas               | Captura específica por plataforma                                          |
| Vídeo recebido             | `flutter_webrtc` / `RTCVideoRenderer`         | Renderização do video track recebido                                       |
| Adaptação de qualidade     | Controlador próprio sobre estatísticas WebRTC | Ajuste dinâmico de bitrate/qualidade e troca de perfil quando necessário   |
| Telemetria                 | WebRTC statistics + DataChannel               | Feedback de desempenho da conexão e do receptor                            |

---

# Princípios arquiteturais

O projeto deve preservar os seguintes princípios:

1. **Firebase não transporta arquivos, mensagens ou vídeo.**
2. **Uma única `RTCPeerConnection` é reutilizada para DataChannel e mídia.**
3. **Vídeo não entra no buffer/histórico.**
4. **O streaming permanece efêmero.**
5. **A qualidade automática prioriza bitrate/qualidade antes de reduzir resolução.**
6. **O receptor também participa da decisão de qualidade.**
7. **TURN é fallback, não o caminho principal.**
8. **Código específico de plataforma fica isolado na camada de mídia/captura.**
9. **O app continua sendo único: qualquer dispositivo pode atuar como emissor ou receptor.**
10. **Nenhum banco de dados persistente é necessário para o funcionamento normal de uma sessão.**
