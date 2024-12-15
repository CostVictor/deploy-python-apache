# Configuração do Apache com Python no Ubuntu Server

Neste guia, vamos configurar o servidor web Apache para servir uma aplicação Python. Primeiro, configuraremos o Apache e, em seguida, adicionaremos o arquivo WSGI que o Apache usará para executar o projeto.

## Por que usar o Apache?

`Apache` é um servidor web amplamente utilizado para hospedar e gerenciar conteúdo da web. Ele é responsável por receber as solicitações HTTP dos clientes (como navegadores) e fornecer arquivos estáticos, como HTML, CSS e JavaScript. Além disso, o Apache pode executar aplicativos dinâmicos, como aqueles escritos em Python, por meio de módulos adicionais. Ele é conhecido por sua confiabilidade, flexibilidade e segurança.

## Por que usar o WSGI?

`WSGI` (Web Server Gateway Interface) é um padrão que define a interface entre servidores web e aplicativos Python. O WSGI permite que servidores web, como o Apache, comuniquem-se com aplicações Python de forma padronizada. Isso garante que a aplicação Python possa ser executada independentemente do servidor web específico. O WSGI é essencial para garantir que o código Python possa receber e processar solicitações HTTP e enviar respostas de volta através do servidor web.

`mod_wsgi` é um módulo do Apache que implementa o padrão WSGI. Ele atua como um intermediário entre o Apache e a aplicação Python, permitindo que o Apache execute aplicações Python de forma eficiente. O `mod_wsgi` gerencia a comunicação entre o servidor web e a aplicação, cuidando da execução e da entrega de respostas HTTP.

---

## Passo 1: Instalação do Apache e do Python

Primeiro, é necessário instalar o Apache, o módulo WSGI e outras ferramentas essenciais para configurar o servidor e garantir o suporte adequado a aplicativos Python. O comando abaixo instala os pacotes necessários:

```bash
sudo apt install apache2 libapache2-mod-wsgi-py3 python3 python3-dev python3-venv build-essential pkg-config
```

- `apache2`: Servidor web Apache.  
- `libapache2-mod-wsgi-py3`: Módulo WSGI para Python 3, que permite ao Apache executar aplicativos Python.  
- `python3`: Versão do Python 3.  
- `python3-dev`: Fornece cabeçalhos e bibliotecas necessários para compilar extensões Python.  
- `python3-venv`: Ferramenta para criar ambientes virtuais Python, isolando dependências específicas do projeto.  
- `build-essential`: Ferramentas como `gcc`, `g++` e `make`, necessárias para compilar dependências Python com código nativo.  
- `pkg-config`: Localiza bibliotecas instaladas no sistema, usadas por dependências Python que requerem bibliotecas externas.  

## Passo 2: Configuração do Apache

### 1. Criar o Domínio do Projeto

Crie um diretório para o seu domínio dentro de `/var/www/`:

```bash
sudo mkdir -p /var/www/<domínio>/
```

Substitua <domínio> pelo nome do seu domínio (ex: `meusite.com`).

- `-p`: Significa "pais" (do inglês "parents"). Quando usado, faz com que o mkdir crie qualquer diretório pai necessário ao longo do caminho especificado.

### 2. Adicionando seu Projeto Python na Pasta do Domínio

1. Vá para o diretório de seu domínio:

```bash
cd /var/www/<domínio>/
```

2. Faça um clone de seu projeto do GitHub:

```bash
git clone <chave ssh do projeto>
```

- Se você ainda não possui uma chave SSH configurada para seu Ubuntu Server, pode seguir o passo a passo disponível em: [Create Key SSH](https://github.com/CostVictor/create-key-ssh).

3. Crie um ambiente virtual Python para isolar as dependências do projeto:

```bash
python3 -m venv venv
```

- `-m`: é usado para executar um módulo como um script. No caso do venv, ele executa o módulo venv que está incluído no Python para criar um novo ambiente virtual. O uso de -m permite que você execute o módulo diretamente a partir da linha de comando, sem precisar saber o caminho completo do script.

4. Ative o ambiente virtual:

```bash
source venv/bin/activate
```

5. Baixe as dependências (se houver).

Se você tiver um arquivo `requirements.txt` com as dependências do seu projeto, instale-as usando o pip:

```bash
pip install -r requirements.txt
```

- `-r`: este parâmetro indica que o pip deve ler a lista de pacotes a serem instalados a partir de um arquivo. No caso, requirements.txt é o arquivo que lista todas as dependências do projeto.

Caso não possua, instale as dependências manualmente:

```bash
pip install <dependência>
```

**OBSERVAÇÃO**: Se, ao executar seu projeto, você encontrar o erro `ModuleNotFoundError: No module named 'pkg_resources'`, basta executar o comando:

```bash
pip install setuptools
```

## Passo 3: Configurar o Arquivo WSGI para Seu Projeto

Dependendo se você está usando `Flask` ou `Django`, o arquivo WSGI terá configurações diferentes.

### 1. Para um Projeto `Flask`:

No diretório raiz do projeto, crie o arquivo `wsgi.py`:

```bash
sudo nano /var/www/<domínio>/<projeto>/wsgi.py
```

Adicione o seguinte código no arquivo:

```python
from app import app as application

if __name__ == "__main__":
    application.run()
```

### 2. Para um Projeto `Django`:

Com o framework Django, o WSGI já é configurado automaticamente no momento da criação do projeto. Essa configuração pode ser encontrada no arquivo `wsgi.py`, localizado na pasta principal do projeto, contendo algo semelhante a:

```python
import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', '<nome_do_projeto>.settings')

application = get_wsgi_application()
```

## Passo 4: Configuração do Apache para o Domínio

1. Crie um arquivo de configuração Apache para o seu domínio:

```bash
sudo nano /etc/apache2/sites-available/<domínio>.conf
```

2. Atualize o arquivo com as seguintes configurações:

```txt
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName <domínio>
    ServerAlias www.<domínio>

    # AVISO: Apague estas mensagens quando terminar a configuração.

    # Configure o proxy e o proxy reverso do seu projeto para proteger o IP do seu servidor:

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:<porta>/
    ProxyPassReverse / http://127.0.0.1:<porta>/

    # Em 'DocumentRoot' defina a pasta raiz do seu projeto:

    DocumentRoot /var/www/<domínio>/<projeto>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # Por questões de segurança, daremos acesso apenas a pasta pública de seu projeto. Está pasta será disponibilizada pela rede, garanta que não tenha nenhum arquivo sensível dentro dela.

    <Directory /var/www/<domínio>/<projeto>/<pasta pública do seu projeto>>
        Require all granted
    </Directory>

    # Se você possui alguma pasta especifica dentro do dominio que não deve ser acessível, adicione a seguinte configuração para cada uma delas:
    # <Directory /var/www/<domínio>/<Nome da pasta>>
    #   Require all denied
    # </Directory>

    # Se você possui algum arquivo especifico dentro do dominio que não deve ser acessível, adicione a seguinte configuração para cada um deles:
    # <FilesMatch "/<nome do arquivo.extensão>">
    #   Require all denied
    # </FilesMatch>

    WSGIDaemonProcess <nome de variável para sua aplicação> python-path=/var/www/<domínio>/<projeto> python-home=/var/www/<domínio>/<projeto>/venv
    WSGIProcessGroup <nome de variável para sua aplicação>
    WSGIScriptAlias / /var/www/<domínio>/<projeto>/wsgi.py
</VirtualHost>
```

- `Bloco <VirtualHost>`: Configura um host virtual para servir seu site na porta 80.
- `Proxy e Proxy Reverse`: Redireciona todas as solicitações HTTP para o aplicativo rodando localmente, protegendo o IP do servidor e mantendo as sessões do usuário.
- `DocumentRoot`: Define onde os arquivos estáticos são armazenados.
- `Bloco <Directory>`: Configura permissões para o diretório do projeto.
- `WSGIDaemonProcess e WSGIProcessGroup`: Configuram o ambiente WSGI para seu aplicativo Python.
- `WSGIScriptAlias`: Mapeia a URL raiz para o arquivo WSGI que executa seu aplicativo Python.

---

(Opcional) Se você também quiser limitar o acesso a `requirements.txt` e `venv` aos usuários específicos que possuem acesso a seu servidor, você pode executar:

```bash
chmod 700 /var/www/<domínio>/requirements.txt
chmod 700 /var/www/<domínio>/venv
```

`700`: Este número especifica as permissões que você está definindo para a pasta:

- 7 (rwx): O proprietário da pasta tem permissões de leitura (r), gravação (w) e execução (x).
- 0 (---): Os membros do grupo não têm nenhuma permissão.
- 0 (---): Outros usuários (não pertencentes ao grupo e não sendo o proprietário) também não têm nenhuma permissão.

---

3. Ative o Módulo Proxy e WSGI para Utilizar os Recursos:

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod wsgi
```

4. Ative o Novo Arquivo de Configuração e Desative o Site Padrão:

```bash
sudo a2ensite <domínio>.conf
sudo a2dissite 000-default.conf
```

5. Reinicie o Apache para Aplicar as Novas Configurações:

```bash
sudo systemctl restart apache2
```

## Passo 5 (Opcional): Configurar o Arquivo Hosts

O arquivo de hosts é um arquivo de sistema que mapeia endereços IP a nomes de host. Ele permite que você defina como o sistema resolve um nome de domínio para um endereço IP específico.

1. Abra o arquivo `/etc/hosts` para edição:

```bash
sudo nano /etc/hosts
```

2. Adicione a seguinte linha ao arquivo:

```txt
127.0.0.1    <domínio>
```

- Isso permite que você acesse o aplicativo em seu navegador usando http://meusite.local em vez de http://localhost.
- É opcional porque essa configuração é local na máquina, funcionando apenas nela.

## Passo 6 (Opcional): Verificar a Configuração

1. Acesse o endereço configurado no navegador:

```txt
http://<domínio>
```

Se a configuração estiver correta, você verá a página inicial do seu projeto.

<!-- - Caso deseje configurar seu projeto para que também acesse banco de dados, siga o passo a passo em: [Config Mysql e PHPAdmin]() -->
