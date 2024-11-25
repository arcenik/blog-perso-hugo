# Notes blog perso


## Faire un article

Créer un nouvel article

```sh
hugo new content content/post/2024-11-22-bosh-virtualbox-first-step.md
```

Serveur local avec contnu drafts

```sh
hugo serve -D
```

## Theme setup

Thème [Github Style](https://themes.gohugo.io/themes/github-style/)

```sh
git submodule add https://github.com/MeiK2333/github-style.git themes/github-style
cd themes/github-style
git pull
```

## Hebergement Github pages

FIXME: terraform pour le DNS
FIXME: pipeline github action
