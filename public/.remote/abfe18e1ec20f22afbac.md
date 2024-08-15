---
title: Python+FlaskでOpenAIのAPIを使用したWebアプリ構築
tags:
  - Python
  - Flask
  - フロントエンド
  - バックエンド
private: false
updated_at: '2024-08-15T15:50:38+09:00'
id: abfe18e1ec20f22afbac
organization_url_name: null
slide: false
ignorePublish: false
---
本記事では，PythonのFlaskフレームワークを使用して，簡単にWebアプリを作成する過程を記します．
以下のようなflowで，APIを用いた画像生成と，その画像を用いてじゃんけんゲームをするWebアプリ構築を最終目標とします．

1. プロンプトを入力する
2. OpenAI（DALL-E-3）のAPIを使用して画像生成する
3. 生成画像を使用してじゃんけんゲーム

今回はAPI Keyを環境変数として使用するため，安全のためフロントエンドとバックエンドを構築します．

## 目次
1. [Flaskとは](#Flaskとは)
2. [環境](#環境)
3. [Flaskのインストール](#Flaskのインストール)
4. [Flaskを動かしてみる](#Flaskを動かしてみる)

## Flaskとは
[Flask（フラスコ）](https://flask.palletsprojects.com/en/3.0.x/)とは，PythonのWebアプリ構築のためのフレームワークで，小〜中規模のアプリ構築に適しており，簡単にアプリ構築が可能です．
PythonによるWebアプリ構築には，他にもDjnago（ジャンゴ）などがあります．

フロントエンド：ユーザーに表示されるUI部分
バックエンド　：アプリを機能させるためのデータとインフラ構築部分

## 環境
| カテゴリ | 値 |
|-----|-----|
| os | macOS 14.5 |
| python | Python 3.10.9 |

## Flaskのインストール
```sh
$ pip install Flask
```
## Flaskを動かしてみる
実際にFlaskを動かしてみます．
[OpenAIでAPIを取得する際はこちらを参照ください．](https://nicecamera.kidsplates.jp/help/6648/#:~:text=%E7%99%BB%E9%8C%B2%E5%AE%8C%E4%BA%86%E3%81%A7%E3%81%99%E3%80%82-,OpenAI%20API%E3%82%AD%E3%83%BC%E3%81%AE%E5%8F%96%E5%BE%97%E6%96%B9%E6%B3%95,%E3%81%97%E3%81%A6%E3%82%B3%E3%83%94%E3%83%BC%E3%81%97%E3%81%BE%E3%81%99%E3%80%82)
API Keyは環境変数で.envに入れています．
imageやindex.htmlは省略しています．

frontend_main.py
```py
from flask import Flask, render_template, request, redirect, url_for, session, jsonify
import logging
import cv2
import random
import os
import requests
import json
from argparse import ArgumentParser
import threading
import urllib
import logging

'''
アプリケーションの構成
"/" ルート (トップページ:http://localhost:5001/): ユーザーがプロンプトを入力し，生成ボタンを押すページ
"/game" ルート (http://localhost:5001/game/): じゃんけんゲームを実行するページ
"/finish" ルート (http://localhost:5001/finish/): ゲームが終了した後に表示されるページ
"/play" ルート (http://localhost:5001/play/): ゲームのじゃんけんのラウンドを処理し、結果を返すエンドポイント
"/start_game" ルート (http://localhost:5001/start_game/): 画像が選択された後にゲームを開始するページ
'''

BACKEND_URL = os.environ.get("BACKEND_URL", "http://127.0.0.1:8001")
app = Flask(__name__)
app.secret_key = "any_random_string"


@app.route("/game", methods=["GET", "POST"])
def index(img_url=None):
    img_url = request.args.get("img_url")
    logging.info(img_url)
    session["player_hp"] = 3
    session["enemy_hp"] = 3
    session["img_url"] = img_url
    return render_template(
        "index.html",
        player_hp=session["player_hp"],
        enemy_hp=session["enemy_hp"],
        img_url=session["img_url"],
    )


@app.route("/finish", methods=["GET", "POST"])
def finish():
    return render_template("finish.html")


@app.route("/play", methods=["POST"])
def play():
    winner = ""
    player_choice = request.form["choice"]
    choices2num = {"rock": 0, "scissors": 1, "paper": 2}
    num2choices = {0: "rock", 1: "scissors", 2: "paper"}
    enemy_choice = random.choice(["rock", "paper", "scissors"])
    if session["player_hp"] == 1:
        enemy_choice = num2choices[(choices2num[player_choice] + 1) % 3]

    result = determine_winner(player_choice, enemy_choice)

    if result == "player":
        session["enemy_hp"] -= 1
    elif result == "enemy":
        session["player_hp"] -= 1

    if session["player_hp"] <= 0 and session["enemy_hp"] > 0:
        winner = "enemy"
    elif session["enemy_hp"] <= 0 and session["player_hp"] > 0:
        winner = "player"

    return jsonify(
        {
            "player_hp": session["player_hp"],
            "enemy_hp": session["enemy_hp"],
            "player_choice": player_choice,
            "enemy_choice": enemy_choice,
            "result": result,
            "winner": winner,
        }
    )


def determine_winner(player, enemy):
    if player == enemy:
        return "draw"
    if (
        (player == "rock" and enemy == "scissors")
        or (player == "scissors" and enemy == "paper")
        or (player == "paper" and enemy == "rock")
    ):
        return "player"
    else:
        return "enemy"


@app.route("/", methods=["GET", "POST"])
def image_generation():
    '''
    1. フォームの送信:ユーザーがフォームにプロンプトを入力して「生成」ボタンを押す->フォームがサーバーに POST リクエストとして送信される
    2. text 変数の取得:フォームから送信されたキーワードは text 変数に保存される
    3. バックエンドにリクエストを送信:requests.postでバックエンドのAPI (BACKEND_URL) にリクエストが送信される
    4. バックエンドからの応答処理: リクエストに成功(status_code == 200)した場合,生成された画像のURLがレスポンスとして返される
    5. "image_generation.html"テンプレートのレンダリング:
    '''
    images = []
    if request.method == "POST":
        text = request.form["keyword_input"]
        response = requests.post(
            "{}/image_generation?text={}&n_image={}".format(BACKEND_URL, text, 1)
        )
        if response.status_code == 200:
            images = response.json()["image_urls"]
            print("frontendでのimages取得完了", images)

    return render_template("image_generation.html", images=images)


@app.route("/start_game")
def show_image():
    img_url = request.args.get("img_url")
    img_path = "static/imgs/you.png"
    if not img_url:
        img_url = "static/imgs/default.png"
    # save_img(img_url, img_path)
    logging.info(len(img_url))
    threading.Thread(
        target=lambda: requests.post(
            "{}/send_slack?img_url={}".format(BACKEND_URL, urllib.parse.quote(img_url))
        )
    ).start()

    return render_template("start_game.html", img_url=img_url)


def save_img(img_url, img_path):
    response = requests.get(img_url)
    with open(img_path, "wb") as f:
        f.write(response.content)


if __name__ == "__main__":
    parser = ArgumentParser()
    parser.add_argument("--port", "-p", type=int, default=5001)
    parser.add_argument("--host", type=str, default="0.0.0.0")
    args = parser.parse_args()

    app.run(debug=True, host=args.host, port=args.port)

```

image_generation.html
```
<!DOCTYPE html>
<html lang="ja">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>画像生成</title>
    <!-- Bootstrap CSS -->
    <link href="https://maxcdn.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css" rel="stylesheet">
    <style>
        .input-group-lg .form-control {
            height: calc(4rem + 2px);
            font-size: 2rem;
            line-height: 1.5;
        }
        /* ボタン */
        .input-group-lg .input-group-append .btn {
            height: calc(4rem + 2px);
            font-size: 2rem;
            padding: 0 1.5rem;
        }
    </style>
</head>

<body>
    <div class="container mt-5">
        <h1 class="text-center mb-4">キャラクターをうみだそう！</h1>
        <div class="search-container">
            <!-- <img src="{{ url_for('static', filename='imgs/default_yobi.png') }}" alt="左側の画像"> -->
            <form action="/" method="post" class="mb-5 d-flex">
                <div class="input-group input-group-lg">
                    <input id="keyword_input" type="text" name="keyword_input" class="form-control" placeholder="キーワードを入力">
                    <div class="input-group-append">
                        <button type="submit" class="btn btn-primary">生成</button>
                    </div>
                </div>
            </form>
            <!-- <img src="{{ url_for('static', filename='imgs/default_yobi.png') }}" alt="右側の画像"> -->
        </div>

        {% if images %}
        <div class="row justify-content-center">
            <div class="col-md-6 mb-4">
                <form action="/start_game" method="get">
                    <input type="hidden" name="img_url" value="{{ images[0] }}">
                    <button type="submit" style="background: none; border: none; padding: 0;">
                        <img src="{{ images[0] }}" alt="検索結果の画像" class="img-fluid">
                    </button>
                </form>
            </div>
        </div>
        {% endif %}
    </div>

    <!-- Bootstrap JS, Popper.js -->
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.16.0/umd/popper.min.js"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script>
</body>

</html>

```
backend_main.py
```py
from fastapi import FastAPI, UploadFile, File
import openai
import os
import logging
from googletrans import Translator
from typing import Optional
import numpy as np
import requests
import cv2

logging.basicConfig(level=logging.INFO)
openai.api_key = os.environ["OPENAI_API_KEY"]

app = FastAPI()

# slack settings
TOKEN = ""
CHANNEL = ""


# image generation
@app.get("/image_generation/")
@app.post("/image_generation/")
async def image_genaration(text: str, n_image: int) -> str:
    """
    Generate image from text using openai api dalle-2

    Parameters
    ----------
    text : str
        text to generate image from
    n_image : int, optional
        number of images to generate, by default 1

    Returns
    -------
    str
        path to generated image
    """

    if not text:
        return {"image_urls": None}

    translator = Translator()
    text_en = translator.translate(text, src="ja", dest="en").text
    logging.info(text_en)

    character = text_en
    condition = (
        ", whole body, left side view, sticker art, white plain back ground, high quality, no words"
    )
    prompt = character + condition
    logging.info(prompt)

    n_image = 1
    response = openai.Image.create(prompt=prompt, n=n_image, size="1024x1024")
    # image_url_1 = response["data"][0]["url"]
    return {"image_urls": [image["url"] for image in response["data"]]}


# speech recognition
@app.post("/speech_recognition/")
async def speech_recognition(path: Optional[str] = None, file: UploadFile = None):
    """
    Speech recognition using openai api

    Parameters
    ----------
    path : str, optional
        path to audio path
    file : UploadFile, optional
        audio path, by default None

    Returns
    -------
    str
        transcript of audio path
    """
    logging.info(path)
    if path is not None and os.path.exists(path):
        audio_data = open(path, "rb")
    elif file is not None:
        audio_data = file.file.read()
    else:
        return {"transcript": None}

    transcript = openai.Audio.transcribe("whisper-1", audio_data)
    logging.info(transcript)
    return {"transcript": transcript}

```


![スクリーンショット 2024-08-11 14.12.29.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569996/4c7f1d79-c833-3a9f-4f4b-02e256d5eeb9.png)

![スクリーンショット 2024-08-11 14.14.24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569996/4125e674-c62c-0075-ad6b-c8ae7b8bd7b4.png)
![スクリーンショット 2024-08-11 14.14.36.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569996/89a81aab-2265-5f62-8eca-6097945165e2.png)
![スクリーンショット 2024-08-11 14.14.44.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569996/86d9f1cc-e2c6-f45a-50b2-23082b5e27cd.png)
![スクリーンショット 2024-08-11 14.14.51.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3569996/f20f71e5-1a28-2288-28c9-15f122d150fd.png)
