# 2025.2-Atividades-05-Docker-Django
Atividade avaliativa com Docker e Django

## Objetivo

Nesta atividade, você irá construir dois Dockerfiles diferentes para uma aplicação Django:
1. **Dockerfile de Desenvolvimento**: Com mapeamento de volumes para desenvolvimento local
2. **Dockerfile de Produção**: Com cópia completa dos arquivos para deploy

## Pré-requisitos

- Docker instalado em sua máquina
- Conhecimentos básicos de Docker e comandos Linux
- Editor de texto (VS Code, Vim, Nano, etc.)

## Atividade

### Parte 1: Preparação do Projeto

#### Passo 1: Criar a estrutura de diretórios

```bash
mkdir django-docker-project
cd django-docker-project
mkdir app
```

#### Passo 2: Criar arquivo requirements.txt

Crie um arquivo `requirements.txt` na raiz do projeto com o seguinte conteúdo:

```txt
Django==4.2.7
```

### Parte 2: Dockerfile de Desenvolvimento

#### Passo 3: Criar Dockerfile.dev

Crie um arquivo chamado `Dockerfile.dev` na raiz do projeto com o seguinte conteúdo:

```dockerfile
# Usar Fedora como imagem base
FROM fedora:latest

# Definir diretório de trabalho
WORKDIR /app

# Atualizar sistema e instalar dependências
RUN dnf update -y && \
    dnf install -y fish python3 python3-pip python3-devel gcc sqlite && \
    dnf clean all

# Instalar Django
RUN pip3 install Django==4.2.7

# Expor porta 8000
EXPOSE 8000

# Comando padrão
CMD ["python3", "manage.py", "runserver", "0.0.0.0:8000"]
```

#### Passo 4: Construir a imagem de desenvolvimento

```bash
docker build -f Dockerfile.dev -t django-dev .
```

#### Passo 5: Executar container de desenvolvimento com volume mapeado

```bash
docker run -it --rm -p 8000:8000 -v $(pwd):/app django-dev fish
```

**No windows, usar o Power Shell** com o comando abaixo
```bash
docker run -it --rm -p 8000:8000 -v ${PWD}:/app django-dev fish
```

**Explicação dos parâmetros:**
- `-it`: Modo interativo com terminal
- `--rm`: Remove o container ao sair
- `-p 8000:8000`: Mapeia a porta 8000 do container para a porta 8000 do host
- `-v $(pwd)/app:/app`: Mapeia o diretório local `app` para `/app` no container
- `fish`: Inicia um terminal (shell)

### Parte 3: Criar a Aplicação Django

#### Passo 6: Dentro do container, criar o projeto Django

```bash
django-admin startproject myproject .
```

#### Passo 7: Criar uma aplicação Django

```bash
python3 manage.py startapp webapp
```

#### Passo 8: Configurar o banco de dados SQLite3

O Django já vem configurado para usar SQLite3 por padrão. Verifique o arquivo `myproject/settings.py`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

#### Passo 9: Adicionar a aplicação ao settings.py

Edite o arquivo `myproject/settings.py` e adicione 'webapp' em `INSTALLED_APPS`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'webapp',  # Adicione esta linha
]
```

#### Passo 10: Configurar ALLOWED_HOSTS

No arquivo `myproject/settings.py`, configure o ALLOWED_HOSTS para aceitar todas as conexões:

```python
ALLOWED_HOSTS = ['*']
```

#### Passo 11: Executar as migrações do banco de dados

```bash
python3 manage.py migrate
```

Este comando criará o arquivo `db.sqlite3` com as tabelas necessárias.

#### Passo 12: Criar superusuário (admin)

```bash
python3 manage.py createsuperuser
```

Quando solicitado, preencha:
- **Username**: admin
- **Email**: (pode deixar em branco ou colocar um email qualquer)
- **Password**: 321
- **Password (again)**: 321

**Nota**: Você receberá um aviso de que a senha é muito curta e comum. Digite `y` para confirmar.

#### Passo 13: Criar uma view simples

Edite o arquivo `webapp/views.py`:

```python
from django.http import HttpResponse

def home(request):
    return HttpResponse("Olá! Bem-vindo à aplicação Django em Docker!")
```

#### Passo 14: Configurar URLs da aplicação

Crie o arquivo `webapp/urls.py`:

```python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.home, name='home'),
]
```

#### Passo 15: Configurar URLs do projeto

Edite o arquivo `myproject/urls.py`:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', include('webapp.urls')),
]
```

#### Passo 16: Executar o servidor de desenvolvimento

```bash
python3 manage.py runserver 0.0.0.0:8000
```

#### Passo 17: Testar a aplicação

Abra o navegador e acesse:
- **Página inicial**: http://localhost:8000
- **Admin**: http://localhost:8000/admin (use username: admin, password: 321)

### Parte 4: Dockerfile de Produção

#### Passo 18: Sair do container de desenvolvimento

Pressione `Ctrl+C` para parar o servidor e digite `exit` para sair do container.

#### Passo 19: Criar Dockerfile.prod

Crie um arquivo chamado `Dockerfile.prod` na raiz do projeto:

```dockerfile
# Usar Fedora como imagem base
FROM fedora:latest

# Definir diretório de trabalho
WORKDIR /app

# Atualizar sistema e instalar dependências
RUN dnf update -y && \
    dnf install -y python3 python3-pip python3-devel gcc sqlite && \
    dnf clean all

# Copiar arquivo de requirements
COPY requirements.txt .

# Instalar dependências Python
RUN pip3 install --no-cache-dir -r requirements.txt

# Copiar todos os arquivos do projeto
COPY app/ ./

# Expor porta 8000
EXPOSE 8000

# Comando para executar o servidor
CMD ["python3", "manage.py", "runserver", "0.0.0.0:8000"]
```

#### Passo 20: Criar script de inicialização (opcional para produção)

Para facilitar a criação do superusuário em produção, crie um arquivo `app/init.sh`:

```bash
#!/bin/bash

# Executar migrações
python3 manage.py migrate --noinput

# Criar superusuário se não existir
python3 manage.py shell <<EOF
from django.contrib.auth import get_user_model
User = get_user_model()
if not User.objects.filter(username='admin').exists():
    User.objects.create_superuser('admin', 'admin@example.com', '321')
    print('Superusuário criado com sucesso!')
else:
    print('Superusuário já existe.')
EOF

# Iniciar servidor
python3 manage.py runserver 0.0.0.0:8000
```

#### Passo 21: Modificar Dockerfile.prod para usar o script (opcional)

Se você criou o script de inicialização, atualize o Dockerfile.prod:

```dockerfile
# Usar Fedora como imagem base
FROM fedora:latest

# Definir diretório de trabalho
WORKDIR /app

# Atualizar sistema e instalar dependências
RUN dnf update -y && \
    dnf install -y python3 python3-pip python3-devel gcc sqlite && \
    dnf clean all

# Copiar arquivo de requirements
COPY requirements.txt .

# Instalar dependências Python
RUN pip3 install --no-cache-dir -r requirements.txt

# Copiar todos os arquivos do projeto
COPY app/ ./

# Dar permissão de execução ao script
RUN chmod +x init.sh

# Expor porta 8000
EXPOSE 8000

# Comando para executar o script de inicialização
CMD ["./init.sh"]
```

#### Passo 22: Construir a imagem de produção

```bash
docker build -f Dockerfile.prod -t django-prod .
```

#### Passo 23: Executar container de produção

```bash
docker run -d -p 8080:8000 --name django-app-prod django-prod
```

**Explicação dos parâmetros:**
- `-d`: Executa em modo detached (background)
- `-p 8080:8000`: Mapeia a porta 8000 do container para a porta 8080 do host
- `--name django-app-prod`: Nomeia o container

#### Passo 24: Verificar logs do container

```bash
docker logs django-app-prod
```

#### Passo 25: Testar a aplicação em produção

Abra o navegador e acesse:
- **Página inicial**: http://localhost:8080
- **Admin**: http://localhost:8080/admin (use username: admin, password: 321)

### Parte 5: Comandos Úteis

#### Listar containers em execução

```bash
docker ps
```

#### Parar o container de produção

```bash
docker stop django-app-prod
```

#### Remover o container de produção

```bash
docker rm django-app-prod
```

#### Acessar o shell de um container em execução

```bash
docker exec -it django-app-prod /bin/bash
```

#### Visualizar logs em tempo real

```bash
docker logs -f django-app-prod
```

## Relatório Passo a Passo

### Resumo das Diferenças entre Desenvolvimento e Produção

| Aspecto | Desenvolvimento (Dockerfile.dev) | Produção (Dockerfile.prod) |
|---------|----------------------------------|----------------------------|
| **Mapeamento de arquivos** | Volume mapeado (`-v`) | Arquivos copiados (`COPY`) |
| **Uso** | Desenvolvimento local com hot-reload | Deploy final da aplicação |
| **Persistência** | Alterações refletem imediatamente | Requer rebuild para mudanças |
| **Execução** | Modo interativo (`-it`) | Modo detached (`-d`) |
| **Porta** | 8000 | 8080 (exemplo) |

### Checklist de Conclusão

- [ ] Dockerfile.dev criado com base em Fedora
- [ ] Dockerfile.prod criado com base em Fedora
- [ ] Container de desenvolvimento usa volume mapeado (-v)
- [ ] Container de produção usa COPY para arquivos
- [ ] Python e Django instalados
- [ ] Projeto Django criado (myproject)
- [ ] Aplicação Django criada (webapp)
- [ ] SQLite3 configurado (padrão)
- [ ] Migrações executadas
- [ ] Superusuário criado (username: admin, password: 321)
- [ ] View simples criada e testada
- [ ] Painel admin acessível e funcional
- [ ] Container de desenvolvimento testado
- [ ] Container de produção testado

## Entrega

Documente todo o processo realizado, incluindo:
1. Screenshots das páginas funcionando (home e admin)
2. Conteúdo dos Dockerfiles criados
3. Comandos utilizados em cada etapa
4. Dificuldades encontradas e como foram resolvidas
5. Diferenças observadas entre os ambientes de desenvolvimento e produção

## Referências

- [Documentação oficial do Django](https://docs.djangoproject.com/)
- [Documentação oficial do Docker](https://docs.docker.com/)
- [Fedora Container Images](https://hub.docker.com/_/fedora)
