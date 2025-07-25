### 🚧 Pre-build stage: instala dependências e prepara assets ###
FROM ruby:3.4.4-alpine3.21 AS pre-builder

# Versões de ferramentas
ARG NODE_VERSION="18.20.3"     # usar versão LTS para compatibilidade
ARG PNPM_VERSION="10.2.0"
ENV PNPM_VERSION=${PNPM_VERSION}
ENV NODE_OPTIONS="--max-old-space-size=4096"

# Configuração de ambiente
ARG RAILS_ENV=production
ENV RAILS_ENV ${RAILS_ENV}
ENV BUNDLE_WITHOUT="development:test"
ENV BUNDLE_PATH="/gems"
ENV BUNDLER_VERSION=2.5.11

# Instala dependências do sistema e ferramentas JS
RUN apk update && apk add --no-cache \
  build-base \
  git \
  curl \
  tzdata \
  postgresql-dev \
  postgresql-client \
  python3 \
  make \
  g++ \
  nodejs \
  npm \
  yarn \
  xz \
  vips \
  && gem install bundler -v ${BUNDLER_VERSION} \
  && npm install -g pnpm@${PNPM_VERSION}

# Define diretório de trabalho
WORKDIR /app

# Instala gems
COPY Gemfile Gemfile.lock ./
RUN bundle config set --local force_ruby_platform true \
  && bundle install -j4 -r3

# Instala dependências JS
COPY package.json pnpm-lock.yaml ./
RUN pnpm install

# Copia todo o restante do código
COPY . .

# Garante diretório de logs (caso RAILS_LOG_TO_STDOUT seja falso)
RUN mkdir -p /app/log

# Precompila os assets em produção
RUN echo "Precompiling assets in $RAILS_ENV..." && \
  if [ "$RAILS_ENV" = "production" ]; then \
    SECRET_KEY_BASE=placeholder RAILS_LOG_TO_STDOUT=enabled bundle exec rake assets:precompile || exit 1; \
    rm -rf spec node_modules tmp/cache; \
  fi

# Salva hash do commit para referência no app
RUN git rev-parse HEAD > /app/.git_sha

# Limpa arquivos desnecessários
RUN rm -rf /gems/ruby/3.4.0/cache/*.gem \
  && find /gems/ruby/3.4.0/gems/ \( -name "*.c" -o -name "*.o" \) -delete \
  && rm -rf .git .gitignore



### ✅ Stage final: imagem menor e pronta para produção ###
FROM ruby:3.4.4-alpine3.21

# Variáveis de ambiente
ARG PNPM_VERSION="10.2.0"
ARG RAILS_ENV=production
ENV RAILS_ENV ${RAILS_ENV}
ENV BUNDLE_WITHOUT="development:test"
ENV BUNDLE_PATH="/gems"
ENV EXECJS_RUNTIME="Disabled"
ENV BUNDLER_VERSION=2.5.11

# Instala apenas o necessário
RUN apk update && apk add --no-cache \
  build-base \
  git \
  openssl \
  tzdata \
  postgresql-client \
  imagemagick \
  vips \
  && gem install bundler -v ${BUNDLER_VERSION}

# Copia runtime necessário do builder
COPY --from=pre-builder /gems/ /gems/
COPY --from=pre-builder /app /app
COPY --from=pre-builder /app/.git_sha /app/.git_sha

WORKDIR /app

EXPOSE 3000

# Ponto de entrada deve ser definido em docker-compose ou ENTRYPOINT script externo
