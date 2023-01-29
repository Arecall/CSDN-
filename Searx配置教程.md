```
#以下全部内容是一个整体，请修改域名后一起复制到SSH运行！

#http访问，该配置不会自动签发SSL
echo "search.shbya.com {
 gzip
 proxy / 127.0.0.1:8888 {
    header_upstream Host {host}
    header_upstream X-Real-IP {remote}
    header_upstream X-Forwarded-For {remote}
    header_upstream X-Forwarded-Port {server_port}
    header_upstream X-Forwarded-Proto {scheme}
  }
}" > /usr/local/caddy/Caddyfile

#https访问，该配置会自动签发SSL，请提前解析域名到VPS服务器
echo "search.shbya.com {
 gzip
 tls admin@moerats.com
 proxy / 127.0.0.1:8888 {
    header_upstream Host {host}
    header_upstream X-Real-IP {remote}
    header_upstream X-Forwarded-For {remote}
    header_upstream X-Forwarded-Port {server_port}
    header_upstream X-Forwarded-Proto {scheme}
  }
}" > /usr/local/caddy/Caddyfile
```