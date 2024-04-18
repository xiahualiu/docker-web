FROM ubuntu:latest

RUN apt-get update
RUN apt-get install git golang-1.20 -y
RUN update-alternatives --install /usr/local/bin/go go /usr/lib/go-1.20/bin/go 1
RUN CGO_ENABLED=1 GOBIN=$HOME/go go install -tags extended github.com/gohugoio/hugo@latest

CMD $HOME/go/hugo
 
