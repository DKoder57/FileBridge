# Roadmap — FileBridge

## Core (obrigatório antes de qualquer submissão às lojas)

- [ ] Módulo de sessão — geração e exibição do código aleatório
- [ ] Módulo de sinalização — handshake WebRTC via Firebase Realtime DB
- [ ] Módulo de transferência — envio/recebimento via DataChannel, fragmentação, indicador de progresso
- [ ] Módulo de buffer — fila circular com auto-exclusão (arquivo + registro) no overflow
- [ ] UI de chat — mensagens de texto e preview de arquivos em uma única tela

## Lançamento nas lojas

- [ ] Build Android → Google Play (Acesso antecipado)
- [ ] Build Windows → Microsoft Store (requer empacotamento MSIX — ver `DECISIONS.md`)
- [ ] Build iOS → App Store
- [ ] Build desktop Linux

## Pós-lançamento

- [ ] Código QR como alternativa à digitação do código de sessão
- [ ] Limite de buffer configurável na tela de ajustes
- [ ] Formulário de feedback dentro do app

## Planejado separadamente (fora deste repo)

- [ ] Workflow de GitHub Actions pra auto-importar issues de um CSV (reaproveitado do repo `project-template` do autor)
