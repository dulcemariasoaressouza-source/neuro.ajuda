# app.py
from flask import Flask, render_template, request, jsonify
import sqlite3
import os
from datetime import datetime

app = Flask(__name__)
DB_NOME = "neuroajuda.db"

# Chave da API da Claude (Anthropic). Configure isso como variável de
# ambiente no seu computador ou no serviço onde for hospedar o site.
# NUNCA coloque a chave real direto aqui no código antes de subir pro GitHub!
CHAVE_API = os.environ.get("ANTHROPIC_API_KEY", "")


def conectar_banco():
    conexao = sqlite3.connect(DB_NOME)
    conexao.row_factory = sqlite3.Row
    return conexao


def criar_tabelas():
    conexao = conectar_banco()
    cursor = conexao.cursor()

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS mensagens (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            remetente TEXT NOT NULL,
            texto TEXT NOT NULL,
            data_hora TEXT NOT NULL
        )
    """)

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS diario (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            tipo TEXT NOT NULL,
            conteudo TEXT NOT NULL,
            humor TEXT,
            energia INTEGER,
            data_hora TEXT NOT NULL
        )
    """)

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS rotina (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            titulo TEXT NOT NULL,
            horario TEXT NOT NULL,
            tipo TEXT NOT NULL,
            concluido INTEGER DEFAULT 0
        )
    """)

    conexao.commit()
    conexao.close()


# ---------------- PÁGINAS ----------------

@app.route("/")
def pagina_inicial():
    return render_template("index.html")


@app.route("/app/chat")
def pagina_chat():
    return render_template("chat.html")


@app.route("/app/diario")
def pagina_diario():
    return render_template("diario.html")


@app.route("/app/rotina")
def pagina_rotina():
    return render_template("rotina.html")


@app.route("/app/respirar")
def pagina_respirar():
    return render_template("respirar.html")


@app.route("/app/foco")
def pagina_foco():
    return render_template("foco.html")


# ---------------- API: CHAT ----------------

@app.route("/api/chat/mensagens", methods=["GET"])
def listar_mensagens():
    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute("SELECT * FROM mensagens ORDER BY id ASC")
    linhas = cursor.fetchall()
    conexao.close()

    mensagens = []
    for linha in linhas:
        mensagens.append({
            "id": linha["id"],
            "remetente": linha["remetente"],
            "texto": linha["texto"],
            "data_hora": linha["data_hora"],
        })
    return jsonify(mensagens)


@app.route("/api/chat/enviar", methods=["POST"])
def enviar_mensagem():
    dados = request.get_json()
    texto_usuario = dados.get("texto", "").strip()

    if texto_usuario == "":
        return jsonify({"erro": "Mensagem vazia"}), 400

    agora = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute(
        "INSERT INTO mensagens (remetente, texto, data_hora) VALUES (?, ?, ?)",
        ("usuario", texto_usuario, agora),
    )
    conexao.commit()

    resposta_ia = perguntar_para_ia(texto_usuario)

    cursor.execute(
        "INSERT INTO mensagens (remetente, texto, data_hora) VALUES (?, ?, ?)",
        ("ia", resposta_ia, agora),
    )
    conexao.commit()
    conexao.close()

    return jsonify({"resposta": resposta_ia})


def perguntar_para_ia(mensagem_usuario):
    if CHAVE_API == "":
        return (
            "Ainda não configurei minha chave de IA. Peça para quem cuida "
            "do site adicionar a variável ANTHROPIC_API_KEY."
        )

    import requests

    instrucoes = (
        "Você é a IA do NeuroAjuda, um espaço de apoio para pessoas com "
        "Autismo, TDAH, Superdotação e outras neurodivergências. Fale de "
        "forma acolhedora, sem julgamentos, sem pressa e sem cobranças. "
        "Ajude a pessoa a dividir tarefas grandes em passos pequenos, uma "
        "coisa de cada vez. Nunca substitua acompanhamento profissional; "
        "em situações de emergência, oriente a pessoa a buscar ajuda "
        "médica imediatamente."
    )

    corpo = {
        "model": "claude-sonnet-4-6",
        "max_tokens": 500,
        "system": instrucoes,
        "messages": [{"role": "user", "content": mensagem_usuario}],
    }

    cabecalho = {
        "x-api-key": CHAVE_API,
        "anthropic-version": "2023-06-01",
        "content-type": "application/json",
    }

    try:
        resposta = requests.post(
            "https://api.anthropic.com/v1/messages",
            json=corpo,
            headers=cabecalho,
            timeout=30,
        )
        dados_resposta = resposta.json()
        return dados_resposta["content"][0]["text"]
    except Exception:
        return "Não consegui responder agora. Tenta de novo daqui a pouco?"


# ---------------- API: DIÁRIO ----------------

@app.route("/api/diario", methods=["GET"])
def listar_diario():
    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute("SELECT * FROM diario ORDER BY id DESC")
    linhas = cursor.fetchall()
    conexao.close()

    entradas = []
    for linha in linhas:
        entradas.append({
            "id": linha["id"],
            "tipo": linha["tipo"],
            "conteudo": linha["conteudo"],
            "humor": linha["humor"],
            "energia": linha["energia"],
            "data_hora": linha["data_hora"],
        })
    return jsonify(entradas)


@app.route("/api/diario", methods=["POST"])
def adicionar_diario():
    dados = request.get_json()
    tipo = dados.get("tipo", "texto")
    conteudo = dados.get("conteudo", "").strip()
    humor = dados.get("humor")
    energia = dados.get("energia")

    if conteudo == "":
        return jsonify({"erro": "Conteúdo vazio"}), 400

    agora = datetime.now().strftime("%Y-%m-%d %H:%M:%S")

    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute(
        "INSERT INTO diario (tipo, conteudo, humor, energia, data_hora) VALUES (?, ?, ?, ?, ?)",
        (tipo, conteudo, humor, energia, agora),
    )
    conexao.commit()
    conexao.close()

    return jsonify({"mensagem": "Entrada salva com sucesso"})


@app.route("/api/diario/exportar", methods=["GET"])
def exportar_diario():
    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute("SELECT * FROM diario ORDER BY id ASC")
    linhas = cursor.fetchall()
    conexao.close()

    texto = "MEU DIÁRIO — NeuroAjuda\n\n"
    for linha in linhas:
        texto += f"[{linha['data_hora']}] ({linha['tipo']})\n"
        if linha["humor"]:
            texto += f"Humor: {linha['humor']}\n"
        if linha["energia"]:
            texto += f"Energia: {linha['energia']}/5\n"
        texto += f"{linha['conteudo']}\n"
        texto += "-" * 30 + "\n"

    from flask import Response
    return Response(
        texto,
        mimetype="text/plain",
        headers={"Content-Disposition": "attachment; filename=meu_diario.txt"},
    )


@app.route("/api/diario/<int:id_entrada>", methods=["DELETE"])
def apagar_diario(id_entrada):
    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute("DELETE FROM diario WHERE id = ?", (id_entrada,))
    conexao.commit()
    conexao.close()
    return jsonify({"mensagem": "Entrada apagada"})


# ---------------- API: ROTINA ----------------

@app.route("/api/rotina", methods=["GET"])
def listar_rotina():
    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute("SELECT * FROM rotina ORDER BY horario ASC")
    linhas = cursor.fetchall()
    conexao.close()

    blocos = []
    for linha in linhas:
        blocos.append({
            "id": linha["id"],
            "titulo": linha["titulo"],
            "horario": linha["horario"],
            "tipo": linha["tipo"],
            "concluido": bool(linha["concluido"]),
        })
    return jsonify(blocos)


@app.route("/api/rotina", methods=["POST"])
def adicionar_rotina():
    dados = request.get_json()
    titulo = dados.get("titulo", "").strip()
    horario = dados.get("horario", "")
    tipo = dados.get("tipo", "atividade")

    if titulo == "" or horario == "":
        return jsonify({"erro": "Preencha título e horário"}), 400

    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute(
        "INSERT INTO rotina (titulo, horario, tipo, concluido) VALUES (?, ?, ?, 0)",
        (titulo, horario, tipo),
    )
    conexao.commit()
    conexao.close()

    return jsonify({"mensagem": "Bloco adicionado"})


@app.route("/api/rotina/<int:id_bloco>/concluir", methods=["PUT"])
def concluir_rotina(id_bloco):
    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute("SELECT concluido FROM rotina WHERE id = ?", (id_bloco,))
    linha = cursor.fetchone()

    novo_valor = 0 if linha["concluido"] else 1
    cursor.execute("UPDATE rotina SET concluido = ? WHERE id = ?", (novo_valor, id_bloco))
    conexao.commit()
    conexao.close()

    return jsonify({"concluido": bool(novo_valor)})


@app.route("/api/rotina/<int:id_bloco>", methods=["DELETE"])
def apagar_rotina(id_bloco):
    conexao = conectar_banco()
    cursor = conexao.cursor()
    cursor.execute("DELETE FROM rotina WHERE id = ?", (id_bloco,))
    conexao.commit()
    conexao.close()
    return jsonify({"mensagem": "Bloco apagado"})


if __name__ == "__main__":
    criar_tabelas()
    app.run(debug=True)
