# Plano de Atualização do `fbnet-firecrawl.yaml` (Revisado)

Este plano descreve os passos para atualizar o arquivo `fbnet-firecrawl.yaml` com base no template `docker-compose.yaml`, garantindo compatibilidade com os stacks existentes (`fbnet.yaml`, `fbnet-ia.yaml`) e respeitando as prioridades definidas.

1.  **Análise Comparativa:**
    *   Comparar `fbnet-firecrawl.yaml` (atual) e `docker-compose.yaml` (template) para identificar todas as diferenças (novas env vars, configurações de serviço, portas, comandos, dependências, nomes, redes, `ulimits`, `x-common-env`, etc.).

2.  **Análise de Conflitos e Redundâncias (Ajustada):**
    *   Mapear serviços, nomes de container/hostnames e portas expostas em `../FBNet/fbnet.yaml` e `../FBNet-Docker-Files/fbnet-ia.yaml`.
    *   **Conflitos de Portas:** Identificar conflitos de portas expostas no host. **Prioridade:** Os serviços existentes em `fbnet.yaml` e `fbnet-ia.yaml` terão prioridade. As portas para os serviços do `fbnet-firecrawl.yaml` serão ajustadas para valores *não utilizados* (ex: `firecrawl-api` usará a porta `3005` do host, pois `3002` está em uso por `fbnet-ia-openwebui`).
    *   **Conflitos de Nomes:** Verificar conflitos de nomes de container/hostname na rede `fbnet`. Manter prefixos como `firecrawl-` para clareza.
    *   **Redundância (Redis):** Verificar se um serviço Redis já existe e está acessível na rede `fbnet`. **Decisão:** Como nenhum Redis acessível foi encontrado e para garantir funcionalidade transparente e isolada, **manter o serviço `firecrawl-redis` dedicado** no `fbnet-firecrawl.yaml`.

3.  **Proposta de Modificação para `fbnet-firecrawl.yaml` (Ajustada):**
    *   Incorporar as atualizações do `docker-compose.yaml` (estrutura, env vars, `ulimits`, etc.).
    *   Resolver conflitos de **portas**, atribuindo novas portas aos serviços do Firecrawl conforme necessário (ex: `3005:${INTERNAL_PORT:-3002}` para `firecrawl-api`).
    *   Resolver conflitos de **nomes** (nenhum encontrado, manter prefixos).
    *   Gerenciar o serviço **Redis** mantendo-o dedicado (`firecrawl-redis`).
    *   Garantir que todos os serviços utilizem a rede externa `fbnet`.
    *   Atualizar o `name` do compose para `firecrawl`.

4.  **Revisão e Confirmação:**
    *   Apresentar o conteúdo completo do arquivo `fbnet-firecrawl.yaml` modificado para revisão e aprovação final.
