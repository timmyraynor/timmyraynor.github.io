Sample TOML file


```
bindAddr = "0.0.0.0"
bindPort = 7000 # Bind port
kcpBindPort = 7001 # KCP bind port, optional configuration according to usage requirements
quicBindPort = 7002 # QUIC bind port, optional configuration according to usage requirements
vhostHTTPPort = 48080 # Virtual HTTP port, optional configuration according to usage requirements
vhostHTTPSPort = 48443 # Virtual HTTPS port, optional configuration according to usage requirements

# Allow the range of remote server ports that can be mapped by the penetration service
allowPorts = [ { start = 50000, end = 60000 } ]

auth.method = "token"
auth.token = "1234567" # Custom authorization token, the client needs to provide the correct token to penetrate

webServer.addr = "0.0.0.0"
webServer.port = 7500
# Dashboard username and password, optional, default is empty
webServer.user = "timqin" # Modify this item
webServer.password = "1234567" # Modify this item
```
