Title: Что делать, если в bootstrap navbar перекрывает основной сайт
Date: 2015-04-19 16:25
Authors: sbog
Slug: bootstrap_navbar
Tags: html, css, bootstrap
Lang: ru

На днях делал сайтик, решил использовать популярный ныне bootstrap 3. С
удивлением обнаружил, что если сделать `navbar navbar-default navbar-fixed-top`
то этот самый navbar перекрывает впоследствии остальные данные за ним, как
будто не знает об их существовании. Очевидного решения не нашлось, поэтому
пришлось сделать маленький хак:

`<body style="padding-top:70px;">`

Можно было заоверрайдить CSS, но мне это не понравилось, поэтому было решено
переопределить стиль. Так работает намного лучше.
