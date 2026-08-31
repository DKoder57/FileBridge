# Roadmap

Planejamento macro do projeto, por fases. O detalhamento de cada item vive nas Issues (ver `issues.csv` e convenção `[Fase][Categoria] Título`).

- [ ] **F0 — Setup**: configuração inicial do repositório, workflows de CI/CD e ambiente
- [ ] **F1 — Core**: código de sessão (ANIMAL-NN), tela de código em espelho, sinalização Firebase/WebRTC, regras de produção do Firebase, TTL client-side, transferência via DataChannel (chunking, backpressure, isolate dedicada), buffer circular, UI de chat, apelido local
- [ ] **F2 — Decisões técnicas**: estratégia de fallback quando o P2P falha (TURN ou falha explícita), avaliação de Rust via FFI para chunking/hash, avaliação de compressão opcional de arquivo — todas condicionadas à instrumentação de métricas feita no F1
- [ ] **F3 — Builds e publicação em loja**: empacotamento MSIX (Windows), build e submissão Android (Google Play), build e submissão Windows (Microsoft Store), build e submissão iOS (App Store), build desktop Linux
- [ ] **F4 — Pós-lançamento**: código QR como alternativa ao código digitado, limite de buffer configurável em ajustes, formulário de feedback no app
- [ ] **F5 — Avançado**: compartilhamento de tela em tempo real, reaproveitando a `RTCPeerConnection` já existente (depende da decisão de fallback TURN do F2 estar tomada antes de começar)

> Sem detalhamento excessivo aqui — cada fase vira Issues específicas conforme o desenvolvimento avança. A numeração de fases segue exatamente `issues.csv`; qualquer nova fase deve ser adicionada em ambos os arquivos ao mesmo tempo para evitar divergência.
