# razbor-pr

Скилл для Claude Code: проводит по изменениям в коде так, как их читает опытный разработчик. Он нужен перед мержем, особенно когда код написал агент, а вы хотите понимать, что именно тот сделал и что может сломаться.

Скилл только читает: в вашем репозитории он ничего не меняет. Разбор пишется по-русски.

[English below](#english)

## Что он делает

Вы называете PR, ветку или последние коммиты, а скилл берёт их через `git` и `gh` и идёт по шагам:

1. Объясняет простыми словами, зачем сделаны изменения.
2. Составляет карту чтения: какие файлы открыть и в каком порядке, включая те, что не менялись, но от которых изменённый код зависит.
3. Проходит по вопросам опытного разработчика:
   - не сломалось ли то, что работало;
   - что будет при сбое посередине и при повторном запуске;
   - можно ли безопасно откатить;
   - кто и с какими данными может это вызвать;
   - поймают ли тесты поломку.
4. Делит находки на три корзины: «исправить обязательно», «по желанию», «выглядит опасно, но в порядке, и почему».
5. В важных местах сначала задаёт вам вопрос («что получится, если…»), и только потом даёт ответ.
6. Каждое утверждение помечает, на чём оно держится:
   - `[проверено по коду]`: видел сам, указаны файл и строка;
   - `[вывод из кода]`: следует из устройства кода, но не запускалось;
   - `[общий опыт]`: общее правило, к этому репозиторию не проверено.
7. В конце даёт готовый промт для исправления обязательных пунктов: агент сначала описывает план по каждому пункту и ждёт вашего согласия, потом правит и запускает тесты.

Обязательных пунктов может не быть. Тогда скилл так и скажет, а не станет придумывать замечания.

## Как выглядит находка

Фрагмент из примера (изменение выдумано, весь разбор в [skills/razbor-pr/references/example.md](skills/razbor-pr/references/example.md)):

> **Подумайте:** провайдер принял SMS и поставил в очередь отправки, но ответ потерялся по дороге, и наш вызов закончился таймаутом на 5 секунде. Сколько SMS получит клиент?
>
> **Ответ:** до трёх. Для нас таймаут выглядит как сбой, и мы отправляем то же сообщение снова, хотя первое уже ушло `[вывод из кода: notify/sms.py:8-12]`. Умеет ли провайдер принимать ключ повтора, из репозитория не видно: не проверено.

> **1. Повтор после таймаута может отправить клиенту несколько одинаковых SMS**
> **Где:** `notify/sms.py:8-12`
> **Сценарий:** провайдер принял сообщение, ответ не пришёл за 5 секунд. Клиент получает «Заказ подтверждён» дважды или трижды.
> **Метка:** `[вывод из кода]`
> **Что сделать:** повторять только при ошибках, после которых сообщение точно не принято, либо передавать провайдеру ключ повтора.

## Установка

Нужны `git` и, для разбора PR по номеру, [GitHub CLI](https://cli.github.com/) (`gh`) с выполненным входом. Для веток и коммитов `gh` не нужен. Никаких программ и зависимостей скилл не ставит, bash не требуется: команды обычные `git` и `gh`.

### Копированием папки

Скопируйте папку `skills/razbor-pr` в папку скиллов Claude Code.

macOS и Linux:

```sh
git clone https://github.com/Iturup1606/razbor-pr.git
cp -r razbor-pr/skills/razbor-pr ~/.claude/skills/
```

Windows (PowerShell):

```powershell
git clone https://github.com/Iturup1606/razbor-pr.git
Copy-Item razbor-pr\skills\razbor-pr $env:USERPROFILE\.claude\skills\razbor-pr -Recurse
```

Только для одного проекта: копируйте в `.claude/skills/` внутри проекта вместо домашней папки.

### Как плагин

В Claude Code:

```
/plugin marketplace add Iturup1606/razbor-pr
/plugin install razbor-pr@razbor-pr
```

То же из терминала: `claude plugin marketplace add Iturup1606/razbor-pr`, затем `claude plugin install razbor-pr@razbor-pr`. Манифесты лежат в папке `.claude-plugin`.

## Как пользоваться

Откройте Claude Code в папке репозитория и напишите, например:

- «разбери PR 123»
- «разбери ветку feature/payments»
- «что тут поменялось в последних трёх коммитах»
- «можно ли мержить, что может сломаться»

Хотите идти не одним куском, а с остановками на вопросах: добавьте «веди по шагам».

## Что внутри

| Файл | Зачем |
|---|---|
| `skills/razbor-pr/SKILL.md` | порядок работы, границы, метки |
| `references/git-recipes.md` | команды git и gh для Windows, macOS и Linux |
| `references/reading-order.md` | как составить карту чтения |
| `references/questions.md` | вопросы опытного разработчика и признаки, когда их задавать |
| `references/findings.md` | три корзины и проверка разбора перед выдачей |
| `references/fix-prompt.md` | шаблон промта на исправление |
| `references/report-template.md` | форма разбора |
| `references/example.md` | пример разбора на выдуманном изменении |

## Ограничения

- Скилл не запускает тесты и сборку: это чужой код с вашими правами, поэтому он только читает и предлагает команду, которую запустите вы.
- Разбор делает модель. Метки показывают, где она уверена, а где нет, но вам всё равно стоит сверить важное по указанным файлам и строкам. Это помощник для чтения, а не замена ревью человеком и проверкам в CI.
- Большие изменения (больше 800 строк или 25 файлов) скилл предложит разбить по темам, а не будет читать поверхностно.
- Команды проверялись на Windows. На macOS и Linux они те же, но там скилл не запускался.

## Лицензия

MIT, см. [LICENSE](LICENSE).

---

## English

**razbor-pr** is a Claude Code skill that walks you through a code change the way an experienced developer reads it. Use it before merging a pull request, especially one written by an agent. The review itself is written in Russian; the skill files are in Russian too.

It is read-only: it never edits files, runs tests, checks out branches or comments on the PR.

What it does, step by step:

1. Takes a PR, a branch or the last few commits through `git` and `gh`.
2. Explains in plain words why the change exists.
3. Builds a reading map: which files to open and in what order, including unchanged files the change depends on.
4. Asks the questions a senior engineer asks: did anything that worked break, what happens on failure and on a second run, can it be rolled back safely, who can call it and with what data, will the tests catch a break.
5. Sorts findings into "must fix", "optional" and "looks dangerous but is fine, and why".
6. At key points asks you a question first and gives the answer afterwards.
7. Labels every claim: verified in the code, inferred from the code, or general experience.
8. Ends with a ready-made prompt that fixes the must-fix items: the agent writes a plan per item and waits for your approval before it edits anything and runs the tests.

Install: copy `skills/razbor-pr` into `~/.claude/skills/` (Windows: `%USERPROFILE%\.claude\skills\`), or add it as a plugin:

```
/plugin marketplace add Iturup1606/razbor-pr
/plugin install razbor-pr@razbor-pr
```

You need `git`, and `gh` (signed in) for PRs by number. No bash is required; the commands are plain `git` and `gh`, written to work in PowerShell, cmd, bash and zsh. They were tested on Windows only. A full worked example is in `skills/razbor-pr/references/example.md`.

MIT licensed.
