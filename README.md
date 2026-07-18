<!--
Para trocar os badges: https://shields.io
-->

# 🌉 FileBridge

Transfere arquivos e mensagens entre dois dispositivos via um código de sessão gerado na hora — sem conta, sem servidor guardando os dados, direto de dispositivo pra dispositivo (P2P).

![Build](https://img.shields.io/github/actions/workflow/status/DKoder57/FileBridge/ci.yml?branch=main)
![License](https://img.shields.io/github/license/DKoder57/FileBridge)
![Last Commit](https://img.shields.io/github/last-commit/DKoder57/FileBridge)

## 📑 Sumário

- [Sobre](#-sobre)
- [Features](#-features)
- [Tecnologias](#️-tecnologias)
- [Como rodar](#️-como-rodar)
- [Estrutura do projeto](#-estrutura-do-projeto)
- [Roadmap](#️-roadmap)
- [Contribuindo](#-contribuindo)
- [Licença](#-licença)

## 📖 Sobre

Documentação técnica completa (arquitetura, fluxo de dados, decisões) vive em [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) e [`docs/DECISIONS.md`](docs/DECISIONS.md) — aqui fica só o essencial pra começar.

## ✨ Features

- [ ] Pareamento por código de sessão (sem QR obrigatório, sem login)
- [ ] Envio de arquivos e mensagens na mesma sessão
- [ ] Transporte P2P criptografado (WebRTC DataChannel, DTLS-SRTP)
- [ ] Buffer circular em memória (sem persistência em disco)
- [ ] Código QR como alternativa ao código digitado (ver [Roadmap](#️-roadmap))

## 🛠️ Tecnologias

| Camada | Tecnologia |
|---|---|
| Frontend | Flutter (Dart) |
| P2P / Transporte | `flutter_webrtc` |
| Sinalização | Firebase Realtime Database |
| Estado | Riverpod |
| Armazenamento local | `path_provider` + fila em memória |

## ▶️ Como rodar

```bash
# Clonar
git clone https://github.com/DKoder57/FileBridge.git
cd FileBridge/app

# Instalar dependências
flutter pub get

# Rodar em desenvolvimento
flutter run
```

Requer um projeto Firebase próprio (plano gratuito Spark) com Realtime Database habilitado — configure `google-services.json` (Android) / `GoogleService-Info.plist` (iOS) seguindo a [documentação do FlutterFire](https://firebase.flutter.dev/).

## 📂 Estrutura do projeto

```
.
├── .github/          # CI (GitHub Actions) e Dependabot
├── docs/             # Arquitetura, roadmap e decisões técnicas
├── issues.csv        # Backlog inicial, importado como GitHub Issues
└── app/
    └── lib/
        ├── core/          # session, signaling, transfer, buffer
        └── features/      # home, chat, feedback
```

## 🗺️ Roadmap

Planejamento macro por fases em [`docs/ROADMAP.md`](docs/ROADMAP.md). O detalhamento de cada item vira Issue no GitHub.

## 🤝 Contribuindo

Projeto pessoal/solo — sugestões e issues são bem-vindas, mas não há processo formal de contribuição externa no momento.

## 📄 Licença

Distribuído sob a licença MIT. Veja [`LICENSE`](LICENSE) para mais detalhes.
