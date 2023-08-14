# streamlit on lambda

## poetryで環境構築

以下リンクを参考に実装した(#Poetryをダウンロードしたpythonと別のバージョンを使用する場合先にvenvの作成が必須というのがややこしい)。
https://zenn.dev/zenizeni/books/a64578f98450c2/viewer/c6af80

## streamlit起動の確認

~~~
poetry add streamlit
poetry run streamlit hello
~~~

## Dockerfileでの実行

以下コマンドを実行し、requirement.txtをapp配下に出力
~~~
poetry export -f requirements.txt --output app/requirements.txt
~~~


~~~
FROM public.ecr.aws/docker/library/python:3.8.12-slim-buster
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.0 /lambda-adapter /opt/extensions/lambda-adapter
# Fastapi公式の起動方法に合わせるため、PORT番号を環境変数で記載しない
# ENV PORT=8000
WORKDIR /var/task
COPY requirements.txt ./
# gccの不測エラーがでるためおまじない的に以下を記載
RUN apt-get update && apt-get install -y build-essential
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt
CMD ["streamlit", "hello"]
~~~

~~~
docker build --no-cache -t steamlitimage .
docker run --name steamlitcontainer -p 8000:8000 steamlitimage
~~~
アプリが起動したが、URLにアクセスできず。。。



以下を参考にport番号8080を指定

https://book.st-hakky.com/docs/data-science-streamlit-configuration/

~~~
FROM public.ecr.aws/docker/library/python:3.8.12-slim-buster
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.0 /lambda-adapter /opt/extensions/lambda-adapter
# Fastapi公式の起動方法に合わせるため、PORT番号を環境変数で記載しない
# ENV PORT=8000
WORKDIR /var/task
COPY requirements.txt ./
# gccの不測エラーがでるためおまじない的に以下を記載
RUN apt-get update && apt-get install -y build-essential
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt
CMD ["streamlit", "hello", "--server.port", "8080"]
~~~


そして、http://localhost:8080/にアクセスするとみることができた







https://zenn.dev/shake_sanma/articles/1c6475ba73da48

検証手順

template yamlとappを一緒にする→完了
Dockerファイルを一緒にする→完了

ローカル実行完了→完了
Dockerでstreamlitローカル起動できるようにする→完了

URLとportを指定してデプロイ
デプロイする