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

# COPY *.py ./
# ここのがコピーされていない気がする→/まで含めるとディレクトリがコピーされる
# COPY templates/ ./templates
# Fastapi公式の起動方法に合わせるため、PORT番号を環境変数で記載しない
# CMD exec uvicorn --port=$PORT main:app
# 参考 https://qiita.com/flutter_daisuki/items/3af8010aa759b3160c35
# CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

# ローカルで実行するときは、以下ではブラウザアクセスできない→Dokcerコンテンのローカルホストで起動するが、外のPCから
# 理由 https://zenn.dev/shake_sanma/articles/1c6475ba73da48
# https://qiita.com/amuyikam/items/01a8c16e3ddbcc734a46

# Portは8080にしないとLambda上で動作しない
# CMD ["uvicorn", "main:app",  "--port", "8080"]