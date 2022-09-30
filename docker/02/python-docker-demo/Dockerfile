FROM python

WORKDIR /home/docker/python-docker-demo

COPY . .

RUN pip3 install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

ENTRYPOINT ["python", "-u", "main.py"]

