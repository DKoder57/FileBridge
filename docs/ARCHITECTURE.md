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
- Sem custo mesmo com milhares de usuários (dados de sinalização são minúsculos)

## Privacidade — o que o Firebase vê

| O Firebase vê | O Firebase NÃO vê |
|---|---|
| Candidatos ICE e SDP (dados de handshake) | Conteúdo dos arquivos transferidos |
| Que uma sessão com código X existe por alguns segundos | Conteúdo das mensagens |
| Nada além disso | Qualquer coisa depois do handshake — a conexão já é P2P |

> ⚠️ **As regras do Firebase deste repositório são apenas para desenvolvimento** (`.read: true / .write: true`). Antes de publicar em produção, restrinja as regras para permitir escrita apenas em novos códigos de sessão e defina TTL de expiração (ver decisão pendente em [`DECISIONS.md`](DECISIONS.md)).

## Estrutura do código

```
app/
└── lib/
    ├── core/
    │   ├── session/      ← geração de código, ciclo de vida da sessão
    │   ├── signaling/    ← canal Firebase + troca SDP/ICE via WebRTC
    │   ├── transfer/     ← DataChannel, fragmentação de arquivos, progresso
    │   └── buffer/       ← fila circular, overflow → apaga arquivo + registro
    ├── features/
    │   ├── home/         ← tela: gerar código OU digitar código
    │   ├── chat/         ← tela: mensagens + anexos de arquivo
    │   └── feedback/     ← tela: formulário de feedback no app
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
