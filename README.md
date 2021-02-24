## My personal website

__Note - I am targeting an arm64 host__

# dev server
```bash
docker run -it --rm \
  -u "$(id -u):$(id -g)" \
  -p 1313:1313 \
  -w "${PWD}" \
  -v "${PWD}":"${PWD}" \
  mbentley/hugo:latest-arm64 server \
  --bind 0.0.0.0
  ```

# build website
```bash
docker run -it --rm \
  -u "$(id -u):$(id -g)" \
  -w "${PWD}" \
  -v "${PWD}":"${PWD}" \
  mbentley/hugo:latest-arm64 \
  -v
  ```

# Serve using nginx
```bash
# From the public directory
docker run \
  -v ${PWD}:/usr/share/nginx/html:ro \
  -p 80:80 \
  -d nginx
```

# Reload live nginx server
```bash
docker exec -it <container> nginx -s reload
```