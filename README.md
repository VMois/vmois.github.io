# VMois personal website

Command to start local container for blog development:

```bash
docker run --rm --name blog --volume="$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll:3 jekyll serve --watch
```

Command to convert PNG/JPEG to WEBP on MacOS:

```bash
cwebp test.jpg -o test.webp
```
