# Arquitetura — FileBridge

## Visão geral

FileBridge é uma ferramenta de transferência **efêmera, ponto-a-ponto, baseada em sessão**. Não é um app de chat persistente, nem armazenamento em nuvem, nem plataforma de mensagens geral.

**Caso de uso de origem:** transferir arquivos e saída de terminal de uma VM Linux para o host, sem habilitar pasta compartilhada e sem estar na mesma rede.

## Como funciona

```
Dispositivo A                    Firebase (signaling)              Dispositivo B
     │                                   │                               │
     │──── gera código de sessão ───────►│                               │
     │                                   │◄──── digita o código ─────────│
     │◄──────── troca SDP/ICE (WebRTC handshake, ~1-2s) ─────────────────►│
     │                                   │                               │
     │═══════════ conexão P2P direta estabelecida, Firebase sai de cena ══│
     │                                   │                               │
     │◄══════════ arquivos e mensagens trafegam direto, criptografados ══►│
```

1. Dispositivo A gera um código de sessão curto e legível (ex: `TIGER-42`)
2. Dispositivo B digita esse código
3. Os dois apps trocam um handshake WebRTC via Firebase Realtime Database (~1–2 segundos)
4. Firebase é descartado. Todo o resto (arquivos + mensagens) flui **direto** entre os dispositivos via WebRTC DataChannel, com criptografia DTLS-SRTP
5. A sessão termina quando qualquer um dos dois fecha o app

## Buffer / histórico

- Cada dispositivo mantém um **buffer circular local** (padrão: 20 itens)
- Itens = mensagens de texto + anexos de arquivo, contados juntos
- No overflow: o item mais antigo é removido da memória **e** seu arquivo é apagado do disco
- Sem banco de dados persistente — tudo em memória, limpo ao fechar o app
- Limite do buffer será configurável em ajustes (roadmap)

## Por que Firebase, por que não P2P puro

- Dois dispositivos em redes diferentes não conseguem se encontrar sem um intermediário de sinalização (problema de NAT — não é solucionável só em software)
- Firebase Realtime Database (plano gratuito Spark) é usado **apenas** para o handshake WebRTC (~3 mensagens, menos de 2 segundos, depois descartado)
- Firebase nunca vê conteúdo de arquivo ou de mensagem — só os candidatos SDP/ICE da negociação WebRTC
- Sem custo mesmo com milhares de usuários (dados de sinalização são minúsculas)

## Privacidade — o que o Firebase vê

| O Firebase vê | O Firebase NÃO vê |
|---|---|
| Candidatos ICE e SDP (dados de handshake) | Conteúdo dos arquivos transferidos |
| Que uma sessão com código X existe por alguns segundos | Conteúdo das mensagens |
| Nada além disso | Qualquer coisa depois do handshake — a conexão já é P2P |

> ⚠️ **As regras do Firebase deste repositório são apenas para desenvolvimento** (`.read: true / .write: true`). Antes de publicar em produção, restrinja as regras para permitir escrita apenas em novos códigos de sessão e defina TTL de expiração (ver decisão pendente em [`DECISIONS.md`](DECISIONS.md)).

## Extensão planejada: compartilhamento de tela

> Ainda não implementado — ver issue de compartilhamento de tela em `issues.csv`. Descrito aqui só para deixar registrado como se encaixa na arquitetura existente, antes de começar a implementação.

O compartilhamento de tela **não abre uma conexão nova nem introduz um servidor de mídia**. Ele reaproveita o mesmo `RTCPeerConnection` já estabelecido no handshake do DataChannel, adicionando um **video track local** (captura de tela) que trafega pela mesma conexão P2P direta — igual acontece hoje com arquivos e mensagens.

```
Dispositivo A                                                    Dispositivo B
     │                                                                  │
     │══ DataChannel já aberto (arquivos/mensagens) ═══════════════════►│
     │                                                                  │
     │── inicia captura de tela ──►│                                    │
     │── renegociação SDP (addTrack) sobre o mesmo PeerConnection ─────►│
     │                                                                  │
     │══ video track (tela) trafega direto, DTLS-SRTP, em paralelo ════►│
     │   ao DataChannel — mesma conexão, sem novo handshake Firebase    │
```

Pontos específicos dessa extensão:

- **Captura de tela é nativa por plataforma** — não existe API cross-platform única no Flutter para isso. Cada plataforma usa seu próprio mecanismo: `MediaProjection` (Android), `desktop_capturer` do `flutter_webrtc` (Windows/Linux), `ReplayKit` com Broadcast Upload Extension (iOS — a mais restrita das quatro, exige target separado no Xcode).
- **Depende diretamente da decisão de fallback TURN** (ver `DECISIONS.md`, decisão em aberto). Vídeo contínuo é muito mais sensível a P2P instável do que a troca pontual de um arquivo — se a sessão só tem STUN e a rede é restritiva, o compartilhamento de tela falha de forma muito mais perceptível do que uma transferência de arquivo que só demora mais.
- **A renegociação SDP pode não precisar reabrir o nó no Firebase.** Como o DataChannel já está aberto entre os dois dispositivos, é possível trocar a nova oferta/resposta SDP direto por ele, sem voltar à sinalização via Firebase — isso ainda precisa ser validado na implementação.

## Estrutura do código

```
app/
└── lib/
    ├── core/
    │   ├── session/      ← geração de código, ciclo de vida da sessão
    │   ├── signaling/    ← canal Firebase + troca SDP/ICE via WebRTC
    │   ├── transfer/     ← DataChannel, fragmentação de arquivos, progresso
    │   ├── buffer/       ← fila circular, overflow → apaga arquivo + registro
    │   └── screen_share/ ← (planejado) video track sobre o PeerConnection existente
    ├── features/
    │   ├── home/         ← tela: gerar código OU digitar código
    │   ├── chat/         ← tela: mensagens + anexos de arquivo
    │   ├── feedback/     ← tela: formulário de feedback no app
    │   └── screen_share/ ← (planejado) tela de visualização do stream recebido
    └── main.dart
```

## Stack técnica

| Camada | Tecnologia | Justificativa |
|---|---|---|
| Framework do app | Flutter (Dart) | Um código-fonte → Android, iOS, Windows, Linux, macOS; publicável em todas as lojas |
| Transferência P2P | `flutter_webrtc` | WebRTC DataChannel para arquivos e mensagens, criptografia DTLS-SRTP nativa |
| Sinalização | Firebase Realtime Database | Plano gratuito (Spark), uso efêmero, sem cartão de crédito |
| Armazenamento local | `path_provider` + fila em memória | Sem necessidade de SQLite/banco; arquivos em diretório temporário, metadados em memória |
| Gerenciamento de estado | Riverpod | Padrões reativos já familiares de outros projetos do autor |
| Compartilhamento de tela (planejado) | `flutter_webrtc` (video track) + captura nativa por plataforma | Mesma lib de WebRTC já usada para o DataChannel; captura em si não tem API única cross-platform, exige código nativo por SO |
