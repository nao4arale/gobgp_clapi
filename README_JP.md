# gobgp_clapi_server
GoBGPと一緒に動かすGoで書かれたAPIです。
何をするかっていうと、要はどっかからこのAPIにjsonでGoBGPのCLIを値に入れて投げると、そのCLIが投入されます。
SSHでわざわざ入ったり、泣きながらexpectを書いたり、公式でAPIが実装していない機能を外部連携したい時に。
遠隔操作なんで一応認証基盤として[JWT](https://jwt.io/)もいれておきました。

# gobgp_clapi_lient
Tokenコードとか、BGPのコマンドとかを手打ちで打つなんて人間のやる仕事じゃないので、
そんなときはGoで書かれたこのクライアントソフトでやれば一連の動作が対話式でちょちょいと出来ます。
今のコードではBGP ipv4 flowspecだけサポートなう。

## 用意するもの
- [Golang](https://golang.org/) (You may use later 1.7)
- [Go BGP](https://github.com/osrg/gobgp/releases/latest).
- [Throw Go BGP CLI](https://github.com/osrg/gobgp/blob/master/docs/sources/cli-command-syntax.md)

## curlを使った例

```bash
root@ubu-client:~# curl -u user:pass -v  http://localhost:3000/api/token
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 3000 (#0)
* Server auth using Basic with user 'user'
> GET /api/token HTTP/1.1
> Host: localhost:3000
> Authorization: Basic dXNlcjpwYXNz
> User-Agent: curl/7.47.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< Date: Sun, 27 Aug 2017 10:14:26 GMT
< Content-Length: 154
< Content-Type: text/plain; charset=utf-8
< 
{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZG1pbiI6dHJ1ZSwibmFtZSI6IkFkbyBLdWtpYyJ9.qsKN2OIk6AW4O4PMgLjyeBYx0BCG7Iopvei-fNuUivo", "expired":"24"}
* Connection #0 to host localhost left intact

root@ubu-client:~# curl -H 'Authorization:Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZG1pbiI6dHJ1ZSwibmFtZSI6IkFkbyBLdWtpYyJ9.qsKN2OIk6AW4O4PMgLjyeBYx0BCG7Iopvei-fNuUivo' -v  -X POST -d '{"command":"/root/go/bin/gobgp global rib add -a ipv4 10.0.0.1/32 community 100:100 med 10 origin igp local-pref 2000"}' http://localhost:3000/api/command
Note: Unnecessary use of -X or --request, POST is already inferred.
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 3000 (#0)
> POST /api/command HTTP/1.1
> Host: localhost:3000
> User-Agent: curl/7.47.0
> Accept: */*
> Authorization:Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZG1pbiI6dHJ1ZSwibmFtZ* upload completely sent off: 119 out of 119 bytes
< HTTP/1.1 200 OK
< Date: Sun, 27 Aug 2017 10:19:11 GMT
< Content-Length: 0
< Content-Type: text/plain; charset=utf-8
< 
* Connection #0 to host localhost left intact
SI6IkFkbyBLdWtpYyJ9.qsKN2OIk6AW4O4PMgLjyeBYx0BCG7Iopvei-fNuUivo
> Content-Length: 119
> Content-Type: application/x-www-form-urlencoded
```
```bash
root@ubu-gobgpd:~# /root/go/bin/gobgp global rib -a ipv4
   Network              Next Hop             AS_PATH              Age        Attrs
*> 10.0.0.1/32          0.0.0.0                                   00:02:01   [{Origin: i} {Med: 10} {LocalPref: 2000} {Communities: 100:100}]
```
## Examples(Using go_gobgp_client)

```bash
root@ubu-client:/go_gobgp_api/go_gobgp_client# go run main.go 

#########################
  Gobgp Flowspec client
#########################

Do you want to do?(add/del): add
destination_ip(MUST): 192.168.0.1/32
source_ip(MUST): 10.0.0.1/32
protocols(tcp/udp/any): tcp
destion_port: 80
source_port: 53
Do you want to then?(accept/discard/rate-limit <ratelimit>): accept

##########################
    check the hash key
##########################

 100 / 100 [========================================================] 100.00% 1s
 Check is done.

OK,Current HASH key is not still changed.
Go to Next Process.

######################################################################

    Current Hash Code: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZG1pbiI6dHJ1ZSwibmFtZSI6IkFkbyBLdWtpYyJ9.qsKN2OIk6AW4O4PMgLjyeBYx0BCG7Iopvei-fNuUivo
	 Post Command: gobgp global rib -a ipv4-flowspec add match destination 192.168.0.1/32 source 10.0.0.1/32 protocol tcp  destination-port =='80'  source-port =='53'  then accept

######################################################################

Do you want to POST this command??(y/n): y

####################
  Working is Done.
####################

```

```bash
root@ubu-bgpd:~# /root/go/bin/gobgp global rib -a ipv4-flowspec
   Network                                                                                                      Next Hop             AS_PATH              Age        Attrs
*> [destination:192.168.0.1/32][source:10.0.0.1/32][protocol:==tcp ][destination-port: ==80][source-port: ==53] fictitious                                00:02:07   [{Origin: ?}]
```