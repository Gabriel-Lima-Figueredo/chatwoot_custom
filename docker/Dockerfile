# ======= Stage 1: Pre-builder =======
FROM ruby:3.2.2-slim AS pre-builder

# Versões de ferramentas
ARG NODE_VERSION="18.20.3"     # usar versão LTS para compatibilidade
ARG PNPM_VERSION="10.2.0"
ENV PNPM_VERSION=${PNPM_VERSION}
ENV NODE_OPTIONS="--max-old-space-size=4096"

# Configuração de ambiente
ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV}
ENV BUNDLE_WITHOUT="development:test"
ENV BUNDLE_PATH="/gems"
ENV BUNDLER_VERSION=2.5.11

# Instala dependências do sistema e ferramentas JS
RUN apt-get update && apt-get install -y --no-install-recommends \
  build-essential \
  git \
  curl \
  tzdata \
  libpq-dev \
  postgresql-client \
  python3 \
  make \
  g++ \
  nodejs \
  npm \
  yarn \
  xz-utils \
  libvips \
  ca-certificates \
  && gem install bundler -v ${BUNDLER_VERSION} \
  && npm install -g pnpm@${PNPM_VERSION} \
  && rm -rf /var/lib/apt/lists/*

# Define diretório de trabalho
WORKDIR /app

# Copia Gemfile e Gemfile.lock para cache de bundler
COPY Gemfile Gemfile.lock ./

# Configura bundler e instala gems
RUN bundle config set --local path '/gems' \
  && bundle config set --local force_ruby_platform true \
  && bundle install -j4

# Copia dependências JS
COPY package.json pnpm-lock.yaml ./

# Instala dependências JS via pnpm
RUN pnpm install

# Copia todo o restante do código
COPY . .

# Garante diretório de logs
RUN mkdir -p /app/log

# Precompila assets só em produção
RUN echo "Precompiling assets in $RAILS_ENV..." && \
  if [ "$RAILS_ENV" = "production" ]; then \
    SECRET_KEY_BASE=placeholder RAILS_LOG_TO_STDOUT=enabled bundle exec rake assets:precompile || exit 1; \
    rm -rf spec node_modules tmp/cache; \
  fi

# Salva hash do commit se existir .git para referência no app
RUN if [ -d .git ]; then git rev-parse HEAD > /app/.git_sha; else echo "no-git-sha" > /app/.git_sha; fi

# Limpa arquivos desnecessários de forma segura
RUN rm -rf /gems/ruby/3.2.2/cache/*.gem \
  && [ -d /gems/ruby/3.2.2/gems/ ] && find /gems/ruby/3.2.2/gems/ \( -name "*.c" -o -name "*.o" \) -delete || true \
  && rm -rf .git .gitignore

# ======= Stage 2: Imagem final (produção) =======
FROM ruby:3.2.2-slim

ARG PNPM_VERSION="10.2.0"
ARG RAILS_ENV=production
ENV RAILS_ENV=${RAILS_ENV}
ENV BUNDLE_WITHOUT="development:test"
ENV BUNDLE_PATH="/gems"
ENV EXECJS_RUNTIME="Disabled"
ENV BUNDLER_VERSION=2.5.11

# Instala apenas dependências necessárias para runtime
RUN apt-get update && apt-get install -y --no-install-recommends \
  build-essential \
  git \
  openssl \
  tzdata \
  libpq-dev \
  postgresql-client \
  imagemagick \
  libvips \
  ca-certificates \
  && gem install bundler -v ${BUNDLER_VERSION} \
  && rm -rf /var/lib/apt/lists/*

# Copia gems, app e arquivo .git_sha do pre-builder
COPY --from=pre-builder /gems/ /gems/
COPY --from=pre-builder /app /app
COPY --from=pre-builder /app/.git_sha /app/.git_sha

WORKDIR /app

EXPOSE 3000

# Comando padrão (ajuste conforme sua necessidade)
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
