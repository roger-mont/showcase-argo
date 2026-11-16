# Argo — contracheques baixados dos portais públicos, sem trabalho manual

![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![Django](https://img.shields.io/badge/Django_6-092E20?logo=django&logoColor=white)
![DRF](https://img.shields.io/badge/Django_REST_Framework-A30000?logo=django&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js_14-000000?logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React_18-20232A?logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?logo=tailwindcss&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Playwright](https://img.shields.io/badge/Playwright-2EAD33?logo=playwright&logoColor=white)
![Chrome Extension](https://img.shields.io/badge/Chrome_Extension_MV3-4285F4?logo=googlechrome&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Código privado · vitrine](https://img.shields.io/badge/C%C3%B3digo_privado-vitrine-555555)

🇺🇸 [English version](README.en.md)

## Contexto

Um escritório de advocacia em Manaus atende muitos servidores públicos, aposentados e pensionistas, e quase todo caso começa pelos contracheques do cliente. Antes, alguém da equipe entrava em cada portal público, um cliente por vez e um mês por vez, para baixar os PDFs e guardá-los à mão. O Argo automatiza isso: a equipe cadastra o cliente e os acessos dele aos portais, escolhe o período e o sistema busca os contracheques em segundo plano. Os arquivos ficam organizados por cliente, portal, matrícula e ano. O projeto nasceu de um script de linha de comando que baixava os contracheques de um único portal e virou um sistema web com vários portais.

## Destaques

- **Um provider por portal**: cada portal público (previdência estadual, previdência municipal, portal do servidor estadual e municipal, federal em desenvolvimento) tem o próprio módulo, com um contrato comum. Para incluir um portal novo, basta escrever o módulo dele e registrá-lo.
- **Buscas em segundo plano**: cada acesso selecionado vira um job com progresso por mês (baixados, já existentes, sem contracheque, falhas). A busca pode ser cancelada e, com "Tentar de novo", refaz só o que faltou.
- **Captcha com uma pessoa no circuito**: quando o portal pede captcha, o job pausa e a imagem aparece num diálogo, em qualquer tela do sistema. A pessoa digita o código e o job continua. Se ninguém responder dentro do prazo (5 min por padrão), o job para com um erro claro.
- **Nada é baixado duas vezes**: o mês que já está guardado é pulado. Um acesso com várias matrículas (vários benefícios) gera um PDF por matrícula.
- **Download como a equipe precisa**: um mês, o .zip de uma matrícula, o .zip do cliente inteiro ou "tudo num PDF só".
- **Coleta assistida onde robô não entra**: o portal do servidor só aceita login gov.br, que bloqueia automação. Nesse caso, a pessoa faz o login no próprio navegador e a extensão **Argo Coletor** percorre a tela de contracheques e envia cada PDF ao sistema.
- **Credenciais protegidas**: as senhas dos portais ficam cifradas com Fernet. Os PDFs só saem pela API autenticada, nunca direto pelo servidor web.
- **Login pelo cadastro central do escritório**: a equipe entra com a conta do sistema central de cadastro (Átrio). Quando o acesso de alguém é cortado lá, a sessão no Argo também cai.

## Telas

> Recortes de capturas reais do sistema. Nomes, documentos e matrículas ficaram de fora do recorte ou foram desfocados.

| | |
|---|---|
| ![Indicadores da tela inicial](assets/01-inicio-indicadores.png) | ![Escolha do período da busca](assets/02-periodo-da-busca.png) |
| **Início**: buscas em andamento, aguardando código (captcha), com erro e total de contracheques guardados. | **Nova busca**: período em um clique (12 meses, ano atual, histórico completo) ou mês e ano exatos. |
| ![Portais disponíveis ao cadastrar um acesso](assets/03-portais-por-acesso.png) | ![Erro de login explicado ao usuário](assets/04-erro-explicado.png) |
| **Acesso por portal**: cada provider aparece com o seu modo (busca automática, coleta pela extensão ou "em breve"). | **Erros em linguagem simples**: "o portal recusou o login", com atalho para corrigir o acesso; o detalhe técnico fica recolhido. |
| ![Contracheques organizados por mês](assets/05-contracheques-por-mes.png) | |
| **Arquivos do cliente**: agrupados por portal e matrícula, com .zip ou PDF único por grupo ou do cliente inteiro. | |

## Arquitetura

```mermaid
flowchart LR
    U["Equipe do escritório<br/>(navegador)"] --> NG["nginx"]
    NG -->|"/"| FE["Frontend<br/>Next.js 14 + React"]
    NG -->|"/api"| API["API<br/>Django + DRF"]
    FE -.->|"polling: progresso e captcha"| API

    subgraph BACK["Backend"]
        API --> RUN["Runner de jobs<br/>(threads em segundo plano)"]
        RUN --> REG["Registry de providers"]
        REG --> PV1["Previdência estadual<br/>(HTTP + parsing de HTML)"]
        REG --> PV2["Previdência municipal<br/>(API REST do portal)"]
        REG --> PV3["Portal do servidor (gov.br)<br/>coleta assistida"]
        REG --> PV4["Federal<br/>em desenvolvimento"]
    end

    API --> DB[("PostgreSQL<br/>clientes, acessos cifrados,<br/>jobs, desafios de captcha")]
    API --> FS[("PDFs<br/>cliente / portal / matrícula / ano")]
    PV1 --> PORT["Portais públicos"]
    PV2 --> PORT

    EXT["Extensão Argo Coletor<br/>(Chrome, MV3)"] -->|"envia cada PDF"| API
    EXT --> GOV["Portal do servidor<br/>(login gov.br feito pela pessoa)"]

    API <-->|"troca de código e reverificação"| ATR["Átrio<br/>(cadastro central e login)"]
```

O frontend acompanha tudo por polling: um único provider de estado no app consulta as buscas ativas e, quando um job pede captcha, abre o diálogo onde a pessoa estiver. Em produção, nginx, frontend e backend rodam em containers Docker num servidor próprio, atrás de Cloudflare Tunnel.

## Ciclo de vida de uma busca

```mermaid
stateDiagram-v2
    state "Aguardando código" as AguardandoCodigo
    state "Concluída" as Concluida
    state "Cancelada" as Cancelada

    [*] --> Pendente: busca criada
    Pendente --> Executando: thread inicia
    Executando --> AguardandoCodigo: portal pede captcha
    AguardandoCodigo --> Executando: pessoa digita o código
    AguardandoCodigo --> Erro: prazo esgotado (padrão 5 min)
    AguardandoCodigo --> Cancelada: cancelar
    Executando --> Concluida: todos os meses processados
    Executando --> Erro: login recusado ou portal fora do ar
    Executando --> Cancelada: cancelar
    Concluida --> Pendente: tentar de novo
    Erro --> Pendente: tentar de novo
    Cancelada --> Pendente: tentar de novo
    Concluida --> [*]

    note right of Executando
        Para cada mês do período:
        já guardado → pula
        baixado → salva um PDF por matrícula
        sem contracheque → registra e segue
        falha de um mês → conta e segue
    end note
```

## O captcha passa por uma pessoa

```mermaid
sequenceDiagram
    participant J as Job em segundo plano
    participant API as API Django
    participant T as Tela (qualquer página)
    actor P as Pessoa da equipe

    J->>API: grava o desafio com a imagem e muda o status para aguardando código
    loop a cada 2 s, até o prazo
        J->>API: já tem resposta? a busca foi cancelada?
    end
    T->>API: polling encontra o desafio pendente
    T->>P: abre o diálogo com a imagem
    P->>T: digita o código
    T->>API: envia a resposta
    J->>API: lê a resposta e volta para executando
    J->>J: continua o login ou o próximo mês
```

Como o job e a tela conversam só pelo banco, a busca segue mesmo que a pessoa troque de página, e o mesmo mecanismo serve a qualquer portal que peça captcha.

## Stack

| Camada | Tecnologias |
|---|---|
| Backend | Python, Django 6, Django REST Framework, SimpleJWT, gunicorn |
| Coleta nos portais | requests + BeautifulSoup, Playwright (portal que exige sessão de navegador), pypdf |
| Frontend | Next.js 14 (App Router), React 18, TypeScript, Tailwind CSS, Radix UI, lucide-react |
| Extensão | Chrome Manifest V3, JavaScript |
| Dados | PostgreSQL (SQLite no desenvolvimento local), PDFs em volume |
| Segurança | Fernet (cryptography) para senhas dos portais, JWT curto, login pelo Átrio |
| Infra | Docker Compose, nginx, servidor próprio atrás de Cloudflare Tunnel |

## Decisões técnicas e desafios

- **O loop fica na base, o portal só sabe baixar um mês.** Cada provider implementa `login()` e `baixar_mes()`. Contagem, retomada, cancelamento e persistência ficam na classe base, e a falha de um mês é registrada sem derrubar a busca inteira. O retorno aceita um PDF ou uma lista por matrícula, o que resolveu o caso de aposentados com vários benefícios no mesmo acesso.
- **Jobs em threads, com o estado no banco.** Para o volume do escritório, uma fila com broker seria peso a mais. O job roda numa thread, grava cada passo no banco e a interface lê por polling. O cancelamento é conferido a cada mês e durante a espera do captcha.
- **HTTP direto sempre que possível.** Os fluxos de login e de consulta foram mapeados no próprio portal, e num dos casos lendo o frontend da aplicação para achar a API REST. Navegador headless (Playwright) ficou só para o portal cuja sessão exigia isso. Assim a busca fica mais rápida e quebra menos.
- **Não contornar o gov.br.** O login gov.br bloqueia automação. Em vez de tentar driblar isso, a coleta virou assistida: a pessoa entra no portal pelo próprio navegador e a extensão faz o trabalho repetitivo. Quando a extensão precisa mostrar a senha salva para a pessoa digitar, ela só aparece para quem tem permissão, some da tela em 2 minutos e cada pedido fica registrado no log.
- **Sessão curta e revogável.** O login pelo Átrio usa um código de uso único que o backend troca com a chave de API do sistema, com um `estado` que amarra a ida à volta. O token do Argo vale 15 minutos e, na renovação, o Átrio é consultado de novo. Um aviso assinado com HMAC derruba a sessão em segundos quando o acesso é cortado.
- **Idempotência por restrição no banco.** A unicidade (acesso, matrícula, ano, mês) garante que repetir uma busca nunca duplica arquivos. Há também uma pausa entre meses para não sobrecarregar os portais públicos.

## Meu papel

Sou o autor da maioria dos commits do repositório (6 de 8, sem contar merges). O commit inicial, com a base do sistema, foi feito por outro integrante da equipe. Os meus commits trouxeram:

- o login pelo Átrio de ponta a ponta (backend, telas de login, renovação com reverificação, aviso de revogação e testes automatizados);
- a extensão Argo Coletor e a coleta assistida do portal do servidor, incluindo o endpoint que recebe os PDFs coletados;
- o painel da extensão para o login gov.br, com a senha revelada só sob permissão;
- a sinalização na interface dos portais antigos que saíram do ar;
- o deploy em produção e a documentação de deploy.

---

O código-fonte deste projeto é privado. Este repositório é só uma vitrine, com descrição, diagramas e telas. Contato: [github.com/roger-mont](https://github.com/roger-mont)
