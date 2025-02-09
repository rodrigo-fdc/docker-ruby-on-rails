```markdown
# Início Rápido: Docker Compose e Ruby on Rails

Este guia mostra como usar o **Docker Compose** para configurar e executar
um aplicativo **Rails/PostgreSQL** em ambiente de desenvolvimento.

---

## 1. Definir o projeto

Começaremos configurando os arquivos necessários para construir e executar o aplicativo.

### Estrutura de arquivos

- **Dockerfile.dev**  
  Define como a imagem Docker do aplicativo (serviço `web`) será construída, instalando Ruby, Bundler e demais dependências.

- **docker-compose.yml**  
  Orquestra os serviços do projeto (o banco de dados PostgreSQL e o próprio Rails), definindo volumes, variáveis de ambiente e portas de acesso.

- **Gemfile** e **Gemfile.lock**  
  Especificam as gems que o projeto Rails utiliza.

- **entrypoint.sh**  
  Script que remove possíveis arquivos residuais de PID do servidor Rails antes de iniciar o processo principal (corrige o problema do `server.pid`).

### 1.1 Dockerfile.dev

Crie um arquivo chamado `Dockerfile.dev` (você pode nomeá-lo apenas `Dockerfile` se preferir) com o conteúdo abaixo:

```dockerfile
# syntax=docker/dockerfile:1

FROM ruby:3.4.1

# Instalar dependências do sistema
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
      build-essential \
      libpq-dev \
      libyaml-dev \
      curl \
      nodejs \
      yarn && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/archives

# Criar e definir diretório de trabalho
RUN mkdir /app
WORKDIR /app

# Copiar Gemfile e Gemfile.lock
COPY Gemfile Gemfile.lock ./

# Instalar as gems
RUN bundle install

# Copiar todo o projeto
COPY . .

# Copiar e configurar o entrypoint
COPY entrypoint.sh /usr/bin/
RUN chmod +x /usr/bin/entrypoint.sh
ENTRYPOINT ["entrypoint.sh"]

# Expor a porta 3000
EXPOSE 3000

# Comando final para iniciar o servidor
CMD ["bin/rails", "server", "-b", "0.0.0.0"]
```

> **Observação**  
> Se você mantiver o nome “Dockerfile.dev”, será necessário referenciá-lo explicitamente no `docker-compose.yml`.  
> Caso prefira usar apenas “Dockerfile”, ajuste o `docker-compose.yml` para `build: .`.

### 1.2 Gemfile

Crie um arquivo **Gemfile** com:

```ruby
source 'https://rubygems.org'
gem 'rails', '~> 7.0'
```

> Ajuste a versão do Rails conforme sua necessidade. Se quiser usar Rails 7.2 ou outra variação, adeque a linha acima.

### 1.3 Gemfile.lock

Crie um arquivo **Gemfile.lock** vazio para que o Dockerfile possa copiá-lo e rodar o `bundle install`:

```bash
touch Gemfile.lock
```

### 1.4 entrypoint.sh

Crie o arquivo `entrypoint.sh` que remove um possível arquivo **server.pid** deixado pelo Rails, evitando erros ao reiniciar o servidor:

```bash
#!/bin/bash
set -e

# Remove a potentially pre-existing server.pid for Rails.
rm -f /app/tmp/pids/server.pid

# Então executa o comando principal informado no CMD do Dockerfile.
exec "$@"
```

---

## 2. Configurar o Docker Compose

Crie o arquivo **docker-compose.yml** com a definição de dois serviços: **db** (PostgreSQL) e **web** (Rails).

```yaml
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: mysecretpassword
    volumes:
      - ./tmp/db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  web:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
    ports:
      - "3000:3000"
    depends_on:
      - db
    environment:
      RAILS_ENV: development
      DATABASE_HOST: db
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: mysecretpassword
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"
```

> - **`volumes: - .:/app`**: mapeia todo o diretório local (host) dentro do contêiner.  
> - **`command`**: sobrescreve o `CMD` do Dockerfile, executando o servidor Rails e removendo o PID previamente.  
> - **`ports`**: expõe a porta 3000 do contêiner na porta 3000 do host, permitindo acessar o app em [http://localhost:3000](http://localhost:3000).  
> - O **serviço `db`** expõe a porta 5432, que pode causar conflito se houver outro Postgres local rodando. Se quiser evitar conflito, mude para `"5433:5432"` ou remova completamente a linha `ports:` caso não necessite acessar o banco externamente.

---

## 3. Construindo o Projeto Rails

### 3.1 Criar a imagem base

Antes de criar a aplicação Rails, construa a imagem para o serviço `web`:

```bash
docker compose build
```

### 3.2 Gerar o aplicativo Rails

Agora que a imagem está pronta, use o contêiner para criar o esqueleto do projeto Rails:

```bash
docker compose run --no-deps web rails new . --force --database=postgresql
```

- **`--no-deps`**: não inicia o serviço de banco (`db`).  
- **`rails new . --force --database=postgresql`**: cria a estrutura do app na pasta atual, forçando substituição de arquivos pré-existentes e usando **PostgreSQL** como banco.

> **Dica (Linux)**  
> Se estiver no Linux, pode ser que os arquivos criados fiquem com dono `root`. Ajuste as permissões:
> ```bash
> sudo chown -R $USER:$USER .
> ```

### 3.3 Reconstruir a imagem

Com o novo **Gemfile** gerado pelo `rails new`, volte a reconstruir a imagem para garantir que as dependências atualizadas sejam instaladas:

```bash
docker compose build
```

---

## 4. Conexão com o Banco de Dados

O Rails, por padrão, assume `localhost` para o banco de dados, mas aqui precisamos que ele se conecte ao serviço `db`. Assim, abra o arquivo **`config/database.yml`** (gerado pelo `rails new`) e substitua o conteúdo:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  host: db
  username: postgres
  password: mysecretpassword
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: app_development

test:
  <<: *default
  database: app_test

production:
  <<: *default
  database: app_production
  username: app
  password: <%= ENV["APP_DATABASE_PASSWORD"] %>
```

---

## 5. Iniciando o Aplicativo

Agora, você pode iniciar seu app com:

```bash
docker compose up
```

O Compose iniciará o contêiner `db` (PostgreSQL) e o contêiner `web` (Rails). Se tudo correr bem, verá logs do Postgres e do Rails no seu terminal.

### 5.1 Criar e Migrar o Banco de Dados

Abra outro terminal e rode:

```bash
docker compose run web rails db:create
```

```bash
docker compose run web rails db:migrate
```

Isso criará e migrará o banco de dados dentro do contêiner. Se precisar popular dados, use `rails db:seed` depois.

---

## 6. Acessar a Aplicação

Abra [http://localhost:3000](http://localhost:3000) no seu navegador. Você deve ver a página de boas-vindas do Rails.

---

## 7. Parar a Aplicação

Para parar tudo de forma limpa, use:

```bash
docker compose down
```

---

## 8. Reiniciar a Aplicação

Para rodar novamente, basta:

```bash
docker compose up
```

e seu aplicativo Rails retorna no mesmo estado, com o banco de dados (persistido em `./tmp/db-data`).

---

## 9. Rebuild do Aplicativo

Algumas alterações no `Gemfile` ou no `docker-compose.yml` podem exigir um rebuild. Em geral:

1. **Alterações simples** (ex.: trocar porta local):
   ```bash
   docker compose up --build
   ```
2. **Mudanças de dependências**:
   - Se adicionar/remover gems no Gemfile, talvez precise rodar:
     ```bash
     docker compose run web bundle install
     docker compose up --build
     ```

---

## 10. Gerando Scaffold e Outros Comandos Rails

Para criar rapidamente recursos (models, controllers, views, rotas) usando o *Scaffold* do Rails, você pode rodar:

```bash
docker compose run web rails g scaffold Contact name:string email:string bithdate:date
```

> **Observação**  
> O comando acima gera arquivos que podem ficar com o proprietário `root` se você estiver no Linux. Nesse caso, rode:
> ```bash
> sudo chown -R $USER:$USER .
> ```
> para restaurar a propriedade dos arquivos ao seu usuário atual.

Depois de criar o scaffold, lembre-se de rodar as migrações se necessário:

```bash
docker compose run web rails db:migrate
```

---

## Observações Finais

- Se você tiver um PostgreSQL local rodando na porta **5432**, pode haver conflito. Mude a porta do serviço `db` para `5433:5432` ou outra porta no host para evitar problemas.  
- Em produção, o Dockerfile costuma ser diferente (muitas vezes multiestágio). Mas este exemplo é ideal para desenvolvimento local.  
- Para mais detalhes sobre Docker Compose, consulte a [documentação oficial](https://docs.docker.com/compose/).  
- Para entender mais sobre Docker no Rails, consulte [Rails Guides - Docker](https://guides.rubyonrails.org/getting_started_with_devcontainer.html) (a partir do Rails 7.1 há geração de Dockerfiles por padrão).

---

**Pronto!** Agora você tem um ambiente Docker Compose completo para desenvolver Ruby on Rails com PostgreSQL de forma simples e consistente.
