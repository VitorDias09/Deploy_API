# API de Upload e Download de Arquivos

API REST desenvolvida em Python com Flask para receber arquivos enviados por clientes e disponibilizá-los para download. O projeto utiliza uma organização em rotas, controladores, modelos e módulo de banco de dados.

> **Situação do projeto:** a documentação descreve os endpoints e o comportamento encontrados no código recebido. Antes da publicação, consulte também a seção [Atenção antes do deploy](#atenção-antes-do-deploy), que registra limitações atuais do projeto.

## Objetivo

Disponibilizar dois recursos HTTP:

- receber e salvar arquivos por meio de uma requisição `POST`;
- baixar um arquivo previamente enviado por meio de uma requisição `GET`.

## Tecnologias utilizadas

- **Python 3.9 ou superior** — linguagem da aplicação;
- **Flask** — framework utilizado para disponibilizar a API REST;
- **Werkzeug** — utilitários utilizados pelo Flask, incluindo a validação segura do nome do arquivo;
- **SQLite** — banco de dados inicializado pela aplicação;
- **Gunicorn** — servidor WSGI indicado para executar a aplicação no Render.

## Estrutura do projeto

```text
upload+download/
├── app.py
├── requirements.txt
├── README.md
├── controllers/
│   ├── download_controller.py
│   └── upload_controller.py
├── database/
│   └── db.py
├── models/
│   ├── download_model.py
│   └── upload_model.py
└── routes/
    ├── download_route.py
    └── upload_route.py
```

Outros arquivos e pastas são criados durante a execução, como `uploads/`, `database.db` e `uploads.log`.

## Pré-requisitos

- Python 3.9 ou superior instalado;
- `pip` disponível no terminal;
- Git (necessário para enviar o projeto ao GitHub e conectá-lo ao Render).

## Instalação e execução local

### 1. Abra a pasta do projeto

No terminal, acesse a pasta que contém o arquivo `app.py`:

```bash
cd caminho/para/upload+download
```

### 2. Crie um ambiente virtual

**Windows (PowerShell):**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

**Linux ou macOS:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Se o PowerShell bloquear a ativação do ambiente virtual, consulte as configurações de política de execução do Windows ou utilize o Prompt de Comando:

```bat
.venv\Scripts\activate.bat
```

### 3. Instale as dependências

Com o ambiente virtual ativado, execute:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 4. Inicie a API

```bash
python app.py
```

A aplicação será iniciada pelo servidor de desenvolvimento do Flask e ficará disponível, por padrão, em:

```text
http://127.0.0.1:5000
```

O código atual ativa `debug=True` nesse modo de execução. Esse modo é destinado apenas ao desenvolvimento local e não deve ser utilizado como servidor de produção.

## Endpoints disponíveis

A API possui os dois endpoints abaixo, conforme as rotas registradas em `app.py`.

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/uploads/upload` | Recebe um arquivo enviado no campo `file` do formulário multipart e salva o arquivo na pasta `uploads/`. |
| `GET` | `/downloads/download/<filename>` | Retorna para download o arquivo identificado pelo nome informado na URL. |

### Tipos de arquivo aceitos

O código permite as seguintes extensões: `png`, `jpg`, `jpeg`, `gif`, `pdf` e `txt`.

O controlador define um limite de conteúdo de **30 KB** por requisição. Atenção: a verificação atual utiliza `request.content_length`, que representa o tamanho total do corpo da requisição HTTP (incluindo os dados de `multipart/form-data`), e não somente os bytes do arquivo. Além disso, a mensagem de erro do código ainda informa 15 KB; essa mensagem está inconsistente com o limite configurado e deve ser corrigida no controlador.

### Exemplo 1 — Enviar um arquivo

Envie uma requisição `POST` para:

```text
http://127.0.0.1:5000/uploads/upload
```

O corpo deve utilizar o formato `multipart/form-data`, com o campo de arquivo chamado **`file`**.

Exemplo com cURL no Windows (substitua o caminho pelo arquivo que deseja enviar):

```powershell
curl.exe -X POST -F "file=@C:\caminho\para\arquivo.pdf" http://127.0.0.1:5000/uploads/upload
```

Exemplo com cURL no Linux ou macOS:

```bash
curl -X POST -F "file=@./arquivo.pdf" http://127.0.0.1:5000/uploads/upload
```

Resposta esperada em caso de sucesso (`HTTP 200`):

```json
{
  "mensagem": "Arquivo salvo com sucesso!",
  "arquivo": "arquivo.pdf"
}
```

O nome retornado em `arquivo` é o nome sanitizado utilizado para salvar o arquivo.

### Exemplo 2 — Baixar um arquivo

Depois de enviar o arquivo, utilize o nome retornado pela API na rota de download:

```text
http://127.0.0.1:5000/downloads/download/arquivo.pdf
```

Para salvar a resposta em um arquivo usando cURL:

```bash
curl -OJ http://127.0.0.1:5000/downloads/download/arquivo.pdf
```

Também é possível abrir o endereço no navegador. O servidor envia o arquivo como anexo para download.

### Respostas de erro previstas no upload

| Situação | Código HTTP | Comportamento |
|---|---:|---|
| Campo `file` não enviado | `400` | Retorna JSON informando que nenhum arquivo foi enviado. |
| Nome do arquivo vazio | `400` | Retorna JSON informando que o nome está vazio. |
| Requisição acima do limite configurado | `400` | Retorna JSON informando o limite; a mensagem atual menciona 15 KB, embora a configuração seja 30 KB. |
| Extensão não permitida | `400` | Retorna JSON com a lista de extensões permitidas. |
| Upload concluído | `200` | Retorna JSON com uma mensagem e o nome do arquivo salvo. |

Se o arquivo solicitado para download não existir, o Flask/Werkzeug responde com erro HTTP `404`.

## Arquivos gerados e banco de dados

- **`uploads/`** — diretório onde o controlador salva os arquivos enviados;
- **`uploads.log`** — arquivo de log configurado pelo controlador de upload;
- **`database.db`** — arquivo SQLite criado pela função `init_db()`;
- **tabela `uploads`** — tabela criada na inicialização, com os campos `id`, `filename` e `filepath`.

**Observação sobre o código atual:** a inicialização do banco de dados cria a tabela, e os Models possuem métodos para consultar e gravar registros. Porém, o controlador de upload atualmente salva o arquivo diretamente no sistema de arquivos e não chama `UploadModel.save_file()`. Da mesma forma, a rota de download utiliza diretamente a pasta `uploads/`, sem consultar o banco de dados. Portanto, não se deve interpretar a existência da tabela como garantia de que os uploads estão sendo registrados nela.

## Variáveis de ambiente

O código recebido não exige variáveis de ambiente próprias para funcionar. No Render, a plataforma disponibiliza automaticamente a variável `PORT`, que será utilizada pelo comando de inicialização do serviço.

Não coloque senhas, tokens ou outras informações privadas diretamente no código ou no repositório.

## Publicação no Render

O projeto pode ser publicado como um **Web Service** conectado ao repositório GitHub.

Configurações sugeridas:

| Campo no Render | Valor |
|---|---|
| Runtime / Environment | `Python 3` |
| Root Directory | Deixe em branco se `app.py` e `requirements.txt` estiverem na raiz do repositório. Se estiverem dentro de `upload+download/`, informe essa pasta. |
| Build Command | `pip install -r requirements.txt` |
| Start Command | `gunicorn app:app --bind 0.0.0.0:$PORT --access-logfile - --error-logfile -` |

O primeiro `app` do comando identifica o arquivo `app.py`; o segundo identifica a variável Flask `app` criada dentro desse arquivo. O Gunicorn importa essa aplicação e a executa como servidor WSGI. O endereço `0.0.0.0` permite que o serviço receba conexões externas, e `$PORT` utiliza a porta fornecida pelo Render.

Documentação oficial: [Render — Deploy a Flask App](https://render.com/docs/deploy-flask) e [Render — Disk](https://render.com/docs/disks).

### Atenção antes do deploy

O projeto grava uploads e o banco SQLite no disco local do serviço. Em serviços Render sem disco persistente, o sistema de arquivos é efêmero: arquivos criados durante a execução podem ser perdidos quando a instância é reiniciada ou substituída. Isso afeta principalmente a pasta `uploads/` e o arquivo `database.db`.

Para manter os arquivos entre reinicializações, será necessário configurar armazenamento persistente compatível com o serviço ou adaptar a aplicação para utilizar um serviço externo de armazenamento. Um disco persistente do Render possui disponibilidade e condições próprias de plano; confira a documentação e os custos atuais antes de escolhê-lo.

Além disso, o código utiliza caminhos relativos (`uploads` e `database.db`). A aplicação deve ser iniciada a partir da pasta do projeto, conforme as configurações acima.

## Testes após a publicação

Depois que o Render concluir o deploy, substitua `<URL-DO-SERVICO>` pelo endereço público fornecido pela plataforma.

**Upload:**

```bash
curl -X POST -F "file=@./arquivo.pdf" <URL-DO-SERVICO>/uploads/upload
```

**Download:**

```bash
curl -OJ <URL-DO-SERVICO>/downloads/download/arquivo.pdf
```

Faça primeiro o upload e utilize exatamente o nome retornado pela resposta ao testar o download. Confirme os resultados também pelo Postman ou Insomnia. O deploy só deve ser considerado validado depois que os endpoints forem testados no endereço público.

## Licença

Nenhuma licença foi identificada nos arquivos recebidos. Defina uma licença antes de distribuir ou reutilizar publicamente o projeto, caso isso seja necessário para a atividade.
