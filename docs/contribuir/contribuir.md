Como contribuir com este Wiki
=============================

*Importante:* Este documento ainda está em desenvolvimento e não está completo ainda.

## Sobre este documento

Este documento descreve os passos necessários para contribuição com o Wiki do Tchelinux.

## Visão geral do processo

Para contribuir com o Wiki é necessário clonar o repositório do Github, fazer as alterações, commitar as mudanças e criar um pull request para que as mudanças sejam adicionadas ao Wiki.

## Requisitos

- Ter uma conta no Github
- Possuir conhecimentos em Markdown (linguagem de marcação)
- Saber usar um editor de texto simples (Atom, Sublime, Gedit, Kate, Vim, Emacs, Nano etc) 
- Entendimento básico sobre como a ferramenta Git funciona (repositórios, branches, forks etc)

## Começando 

### Clonando o repositório

```
git clone https://github.com/TcheLinux/Wiki.git
```

### Criando um branch de trabalho

```
cd Wiki

git branch -b nova_pagina
```

### Fazendo alterações

```
cd docs

vim nova_pagina.md

cd .. 

vim navigation.md

```

### Gravando mudanças

```
git add 

git commit -m "Nova pagina no wiki"
```

## Criando um pull request



## Recursos

- [Livro Git Pro v2](https://git-scm.com/book/pt-br/v2)
