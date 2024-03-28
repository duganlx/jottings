# nginx

要将 `/eam/api/xxx` 路由到 `/api/xxx`，在 nginx 的 location 中添加 `rewrite ^/eam/api/(.*)$ /api/$1 break;` 