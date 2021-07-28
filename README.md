# VMois personal website

Command for development:

```bash
docker run --name blog --volume="$PWD:/srv/jekyll" -p 3000:4000 -it jekyll/jekyll:latest jekyll serve --watch --drafts
```
