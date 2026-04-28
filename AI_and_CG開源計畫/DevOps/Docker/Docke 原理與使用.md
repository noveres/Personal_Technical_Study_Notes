Docker 環境需要運行在主機上，這台主機叫（Docker Host），只要安裝了 Docker 環境，Docker 的進程 **Docker Daemon** 就會一直運行，隨時監聽是否需要服務，並提供了客戶端工具 Client (docker-cli)  讓我們能透過指令，去操作 Docker Daemon

 Docker 也提供了一個應用市場( Registry )，將 app 集中在一個應用市場( 如：MySQL、NGINX )，這些可以被包進 Docker 容器的 APP 我們叫做鏡像（Image），簡單來說是應用程式的軟體包，
 
 我們要使用時，只需要透過 `docker pull`這個指令，發給 **Docker Daemon** ，它就會連線到**應用市場**，去下載一個**鏡像**，把鏡像下載到本機，接下來透過  `docker run` 就會基於該映像啟動一個應用程式，我們會把這個用鏡像啟動的應用程式叫做**容器（Container）**
 
 ```Docker
 docker pull 鏡像名稱
 ```

 ```Docker
 docker run 映像名
 ```
 

 我們可以啟用非常多容器，每個容器都是一個應用，如果我們想要自己註冊一個容器，可以使用`docker build`，將應用程式及其環境打包成 Docker 映像檔（Image），是實現「一次撰寫，到處執行」的核心。
 
 ```Docker
 docker build 軟體名稱
 ```


 ![[Pasted image 20260427142631.png]]

小結：**鏡像(Images)** 就是軟體包，用這個軟體包啟動的應用就是**容器（Container）**