FROM public.ecr.aws/docker/library/python:3.8.12-slim-buster
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.0 /lambda-adapter /opt/extensions/lambda-adapter
# Fastapi公式の起動方法に合わせるため、PORT番号を環境変数で記載しない
# ENV PORT=8000
WORKDIR /var/task
COPY requirements.txt ./
COPY main.py ./
# gccの不足エラーがでるため、おまじない的に以下を記載
RUN apt-get update && apt-get install -y build-essential
RUN python -m pip install --upgrade pip
RUN python -m pip install -r requirements.txt
CMD ["streamlit", "run", "main.py", "--server.port", "8080","--server.address=0.0.0.0","--server.enableCORS=false"]

