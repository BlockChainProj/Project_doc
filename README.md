# BlockChainProj 项目文档
## 17308154 王俊焕
### 实验报告
[report.md](https://github.com/BlockChainProj/Project_doc/blob/master/report.md)
### 演示视频
[demo.mp4](https://github.com/BlockChainProj/Project_doc/blob/master/demo.mp4)
### Quick start
- 链端
```shell
cd ~/fisco/node/127.0.0.1
./start_all.sh
pyenv activate python-sdk
```
- 前端
```shell
git clone https://github.com/BlockChainProj/Client.git
cd Client
npm install
npm run dev
```
- 后端
```shell
git clone https://github.com/BlockChainProj/Server.git
cd Server
python -m swagger-server
```
### 前端：基于Vue.js
[BlockChainProj/Client](https://github.com/BlockChainProj/Client)
### 后端：基于python(swagger生成RESTFul API形式的后端代码)
[BlockChainProj/Server](https://github.com/BlockChainProj/Server)
### 实现加分项
友好的前端界面

![](/img/18.png)
