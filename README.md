# streamlit on lambda

streamlitをLambdaで動作させることを試みました。

## poetryで環境構築

以下リンク<https://zenn.dev/zenizeni/books/a64578f98450c2/viewer/c6af80>を参考に作成

※ Poetryをダウンロードしたpythonと別のバージョンを使用する場合先にvenvの作成が必須というのがややこしいかった

## streamlit起動の確認

~~~zsh
poetry add streamlit
poetry run streamlit hello
~~~

## Dockerfileの作成 ローカルで実行

以下コマンドを実行し、requirement.txtをapp配下に出力

~~~zsh
poetry export -f requirements.txt --output app/requirements.txt
~~~

Dockerfileには以下を記載。

~~~zsh
FROM public.ecr.aws/docker/library/python:3.8.12-slim-buster
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.0 /lambda-adapter /opt/extensions/lambda-adapter
WORKDIR /var/task
COPY requirements.txt ./
# gccの不足エラーがでるため、おまじない的に以下を記載
RUN apt-get update && apt-get install -y build-essential
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt
CMD ["streamlit", "hello"]
~~~

以下コマンドを実行。

~~~zsh
docker build --no-cache -t steamlitimage .
docker run --name steamlitcontainer -p 8000:8000 steamlitimage
~~~

アプリが起動したが、URLにアクセスできず。。。

以下リンク<https://book.st-hakky.com/docs/data-science-streamlit-configuration/>を参考にport番号8080を指定

~~~Dockerfile
FROM public.ecr.aws/docker/library/python:3.8.12-slim-buster
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.0 /lambda-adapter /opt/extensions/lambda-adapter
WORKDIR /var/task
COPY requirements.txt ./
# gccの不足エラーがでるため、おまじない的に以下を記載
RUN apt-get update && apt-get install -y build-essential
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt
CMD ["streamlit", "hello", "--server.port", "8080"]
~~~

そして、<http://localhost:8080/>にアクセスするとstreamlitをみることができた

ちなみに、以下リンク：<https://zenn.dev/shake_sanma/articles/1c6475ba73da48>　のように、Dockerでコンテナ内で0.0.0.0にするときもあるが、streamlitで実行時のアドレスを以下のように指定することも可能であった。

~~~Dockerfile
FROM public.ecr.aws/docker/library/python:3.8.12-slim-buster
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.0 /lambda-adapter /opt/extensions/lambda-adapter
WORKDIR /var/task
COPY requirements.txt ./
COPY main.py ./
# gccの不測エラーがでるためおまじない的に以下を記載
RUN apt-get update && apt-get install -y build-essential
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt
CMD ["streamlit", "run", "main.py", "--server.port", "8080","--server.address=0.0.0.0"]
~~~

## deploy

以下リンク<https://github.com/awslabs/aws-lambda-web-adapter/tree/main/examples/fastapi>のfastapiのデプロイを参考にして、templateを作成

~~~template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  python3.8

  Sample SAM Template for FastAPI

# More info about Globals: https://github.com/awslabs/serverless-application-model/blob/master/docs/globals.rst
Globals:
  Function:
    Timeout: 300

Resources:
  FastAPIFunction:
    Type: AWS::Serverless::Function
    Properties:
      PackageType: Image
      MemorySize: 2560
      Events:
        ApiEvents:
          Type: HttpApi
    Metadata:
      Dockerfile: Dockerfile
      DockerContext: ./app
      DockerTag: python3.8-v1

Outputs:
  FastAPIURL:
    Description: "API Gateway endpoint URL for Prod stage for FastAPI function"
    Value: !Sub "https://${ServerlessHttpApi}.execute-api.${AWS::Region}.${AWS::URLSuffix}/"
  FastAPIFunction:
    Description: "FastAPI Lambda Function ARN"
    Value: !GetAtt FastAPIFunction.Arn
  FastAPIIamRole:
    Description: "Implicit IAM Role created for FastAPI function"
    Value: !GetAtt FastAPIFunctionRole.Arn
~~~

samでdeployする

~~~zsh
poetry run sam build
poetry run sam deploy --guided
~~~

## deploy結果

sam deploy時に出力されるURLをcurlしてみると、responseが返却される。
しかし、いざURLにアクセスすると"please wait..."とずっと出力される。

CloudWatch Logsをみると以下の通りになっていて、何度もLambdaが起動停止を繰り返しているように見える

~~~CloudWatch Logs
Collecting usage statistics. To deactivate, set browser.gatherUsageStats to False.
You can now view your Streamlit app in your browser.
URL: http://0.0.0.0:8080
EXTENSION	Name: lambda-adapter	State: Ready	Events: []
START RequestId: c09f53de-0e09-4e3f-96aa-b280d98079c1 Version: $LATEST
END RequestId: c09f53de-0e09-4e3f-96aa-b280d98079c1
REPORT RequestId: c09f53de-0e09-4e3f-96aa-b280d98079c1	Duration: 20.65 ms	Billed Duration: 2678 ms	Memory Size: 2560 MB	Max Memory Used: 196 MB	Init Duration: 2656.67 ms	
START RequestId: 97705915-90e2-42f6-9b2f-9d2199d0f4ff Version: $LATEST
END RequestId: 97705915-90e2-42f6-9b2f-9d2199d0f4ff
REPORT RequestId: 97705915-90e2-42f6-9b2f-9d2199d0f4ff	Duration: 150.06 ms	Billed Duration: 151 ms	Memory Size: 2560 MB	Max Memory Used: 199 MB	
:
:
:
~~~

以下リンク<https://docs.streamlit.io/knowledge-base/deploy/remote-start>を参考に、エラー対策をしたが、改善できなかった。

~~~Dockerfile
FROM public.ecr.aws/docker/library/python:3.8.12-slim-buster
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.0 /lambda-adapter /opt/extensions/lambda-adapter
WORKDIR /var/task
COPY requirements.txt ./
COPY main.py ./
# gccの不足エラーがでるため、おまじない的に以下を記載
RUN apt-get update && apt-get install -y build-essential
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt
# --server.enableCORS=falseを指定しても同様
CMD ["streamlit", "run", "main.py", "--server.port", "8080","--server.address=0.0.0.0","--server.enableWebsocketCompression=false"]

~~~

## まとめ

aws-lambda-adapterを使ってstreamlitを動作させようとしたが、難しかった。