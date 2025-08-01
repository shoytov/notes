# Хэндлим обработку некорректного ввода в ZSH для поднятия настроения

Сегодня хочу рассказать о том, как немного разнообразить времяпрепровождение в консоли, добавив немного юмора, если ваша командная оболочка zsh.

Все, кто работает в терминале (эмуляторе терминала, чтобы меня тут не покусали в комментариях :)), думаю, периодически сталкиваются с тем, что вводят команду неправильно. Например, есть шуточная команда `sl`, которая рисует движущийся поезд, если вы случайно опечатались, когда хотели набрать команду `ls`.
Это служит некой разрядкой и поводом лишний раз улыбнуться. Вот [репозиторий этой утилитки на GitHub](https://github.com/mtoyoda/sl) для любознательных.

А что, если мы хотим, чтобы на ввод любой несуществующей команды, мы получали что-то аналогичное выводу команды `sl`? По умолчанию в ZSH в этом случае выводится сообщение **“command not found”**. Давайте это исправим. 

Для этого нам понадобится:
- непосредственно [zsh](https://github.com/zsh-users/zsh) в качестве командной оболочки;
- [cowsay](https://github.com/cowsay-org/cowsay) - утилита командной строки, которая рисует разные фигурки, которые как бы говорят, наподобие героям комиксов.
- [lolcat](https://github.com/busyloop/lolcat) - утилита для разукрашивания текста градиентом, добавления анимации и т.д.

В ZSH предусмотрена возможность переопределять поведение при возникновении каких-то ситуаций, в том числе, переопределение поведения при возникновении ошибок. В нашем случае нам нужно переопределить вывод, когда команда, вводимая пользователем не найдена. Для этого будем использовать метод `command_not_found_handler`. Добавим в `.zshrc` файл следующий код:

```
command_not_found_handler() {
    cowsay -f tux "LOL! Command not found: $1" | lolcat -a -s 150
    return 127
}
```

Немного пояснений: первая строка будет рисовать там пингвина, говорящего, что введенная нами команда не найдена, пингвин будет появляться построчно (150 - скорость появления). Более подробно с доступными параметрами `lolcat` можно ознакомиться, набрав `man lolcat`. 127 - это код, который zsh отправляет по умолчанию, сохраним это поведение. 

Вот так примерно это выглядит:

![](https://habrastorage.org/webt/ng/sx/wi/ngsxwils2nyvbqeus-bz3gip9uc.gif)

Ну вот, собственно говоря, и все. Мелкие моменты, которые нас окружают в повседневности, делают нас (по крайней мере меня) чуточку счастливее :)

