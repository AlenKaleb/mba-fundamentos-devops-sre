# Exportação anotada — `python-image-main`

Este documento exporta e comenta tecnicamente o código do diretório
`python-image-main/python-image-main`, explicando arquitetura,
conceitos, padrões e boas práticas relevantes.

## Visão geral da arquitetura

- **Tipo de aplicação**: API HTTP simples usando **Flask**.
- **Persistência**: in-memory (`times` e `campeonatos`), sem banco de dados.
- **Empacotamento**: Dockerfile baseado em `python:3.12-slim`.
- **Qualidade**: testes com `pytest` e configuração para SonarCloud.

---

## `src/app.py`

### Código exportado

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

times = []
campeonatos = []

@app.route('/')
def index():
    return "Seja bem-vindo!"

@app.route('/times', methods=['GET', 'POST'])
def handle_times():
    if request.method == 'GET':
        return jsonify(times)
    elif request.method == 'POST':
        time = request.json
        times.append(time)
        return jsonify({'message': 'Time adicionado com sucesso!'})

@app.route('/campeonatos', methods=['GET', 'POST'])
def handle_campeonatos():
    if request.method == 'GET':
        return jsonify(campeonatos)
    elif request.method == 'POST':
        campeonato = request.json
        campeonatos.append(campeonato)
        return jsonify({'message': 'Campeonato adicionado com sucesso!'})

@app.errorhandler(404)
def page_not_found(e):
    return jsonify({'error': 'Página não encontrada'}), 404

if __name__ == '__main__':
    app.run(debug=False, host="0.0.0.0", port='9000')
```

### Anotações técnicas

- **`app = Flask(__name__)`**
  - Inicializa a aplicação Flask. O `__name__` permite que o Flask localize recursos relativos.

- **Estruturas in-memory (`times`, `campeonatos`)**
  - Guardam dados em listas Python. São reiniciadas a cada restart do processo.
  - Ponto de atenção: não há persistência, concorrência ou validação de schema.

- **`@app.route('/')`**
  - Endpoint raiz retorna uma mensagem de boas-vindas. Bom para health check básico.

- **`/times` e `/campeonatos`**
  - Implementam padrão **REST** mínimo com **GET** (listar) e **POST** (criar).
  - `request.json` assume payload válido em JSON; não há validação de entrada.
  - Boas práticas sugeridas: validação com schemas (ex.: Marshmallow/Pydantic) e
    retorno de status code `201` para criação.

- **`@app.errorhandler(404)`**
  - Handler centralizado para erros 404, devolvendo JSON consistente.

- **`app.run(host="0.0.0.0", port='9000')`**
  - Escuta em todas as interfaces para uso em containers.
  - `debug=False` é adequado para produção.

---

## `src/requirements.txt`

### Código exportado

```text
Flask==3.0.3
Werkzeug==3.0.3
pytest==8.3.2
pytest-cov==5.0.0
```

### Anotações técnicas

- **Pinagem de versões**
  - Fixar versões evita “drift” em builds e facilita reprodutibilidade.

- **Dependências de teste no mesmo arquivo**
  - É simples, mas em projetos maiores recomenda-se separar `requirements-dev.txt`.

---

## `src/test_app.py`

### Código exportado

```python
import pytest
from flask import Flask
from app import app 

@pytest.fixture
def client():
    return app.test_client()

def test_index(client):
    response = client.get('/')
    assert response.status_code == 200
    assert response.data.decode('utf-8') == 'Seja bem-vindo!'

def test_get_times(client):
    response = client.get('/times')
    assert response.status_code == 200
    assert response.json == []

def test_create_time(client):
    response = client.post('/times', json={"nome": "Time A"})
    assert response.status_code == 200
    assert response.json['message'] == 'Time adicionado com sucesso!'

    response = client.get('/times')
    assert response.json == [{"nome": "Time A"}]

def test_get_campeonatos(client):
    response = client.get('/campeonatos')
    assert response.status_code == 200
    assert response.json == []

def test_create_campeonato(client):
    response = client.post('/campeonatos', json={"nome": "Campeonato A"})
    assert response.status_code == 200
    assert response.json['message'] == 'Campeonato adicionado com sucesso!'

    response = client.get('/campeonatos')
    assert response.json == [{"nome": "Campeonato A"}]

def test_404_error(client):
    response = client.get('/nonexistent')
    assert response.status_code == 404
    assert response.json['error'] == 'Página não encontrada'

if __name__ == '__main__':
    pytest.main()
```

### Anotações técnicas

- **`app.test_client()`**
  - Cliente de teste do Flask para simular requisições sem subir servidor real.

- **Testes CRUD básicos**
  - Cobrem caminhos felizes de GET/POST e o handler 404.
  - Em APIs reais, incluir testes para payloads inválidos e métodos não suportados.

- **Ponto de atenção**
  - As listas `times` e `campeonatos` são globais e compartilhadas entre testes.
  - Em cenários maiores, recomenda-se setup/teardown para isolamento.

---

## `Dockerfile`

### Código exportado

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY src .

RUN pip install --no-cache -r requirements.txt

EXPOSE 9000

CMD ["python", "app.py"]
```

### Anotações técnicas

- **Imagem base `python:3.12-slim`**
  - Reduz tamanho final do container comparado à imagem completa.

- **`WORKDIR /app`**
  - Define o diretório de trabalho padrão para o container.

- **`COPY src .`**
  - Copia o código e o `requirements.txt`.
  - Boas práticas: copiar `requirements.txt` antes para aproveitar cache do Docker.

- **`RUN pip install --no-cache -r requirements.txt`**
  - Instala dependências sem cache para reduzir tamanho da imagem.

- **`EXPOSE 9000` + `CMD`**
  - Documenta a porta e define o comando padrão.

---

## `settings.yml`

### Código exportado

```yaml
registry: 'manoelmineiro'
repository: 'python-image'
dependencies-path: 'src/requirements.txt'
```

### Anotações técnicas

- **Configuração de pipeline**
  - Parametriza registro, repositório e caminho de dependências.
  - Útil para CI/CD e automações sem hardcoding em scripts.

---

## `sonar-project.properties`

### Código exportado

```properties
sonar.projectKey=manoelmineiro_python-image
sonar.organization=manoelmineiro

# This is the name and version displayed in the SonarCloud UI.
sonar.projectName=python-image
sonar.projectVersion=1.0


# Path is relative to the sonar-project.properties file. Replace "\" by "/" on Windows.
#sonar.sources=.

# Encoding of the source code. Default is default system encoding
#sonar.sourceEncoding=UTF-8

sonar.python.version=3.12
```

### Anotações técnicas

- **Integração com SonarCloud**
  - Define chaves de projeto/organização e versão do Python.
  - Pode ser estendido com `sonar.sources`, `sonar.tests` e cobertura.

---

## Recomendações gerais e boas práticas

- **Persistência real**: substituir listas globais por banco (ex.: SQLite/PostgreSQL).
- **Validação de entrada**: schemas para payloads JSON e status codes adequados.
- **Separar dependências**: `requirements.txt` (prod) e `requirements-dev.txt` (dev/test).
- **Observabilidade**: logs estruturados e health checks específicos.

