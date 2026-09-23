# Приручить `angular.json`

🟠 Уровень: средний · ⏱ 60–90 минут · Тема: [start](../../lecture/advanced.md)

### Как запускать

Терминальная задача. Работаем в приложении `task-board`, созданном
в [задаче 1](https://github.com/DaryaVkh/angular-course-start-task).

```bash
cd ~/angular-playground/task-board
```

Если задача 1 не сделана — создайте проект заново:

```bash
ng new task-board --style=scss --routing --ssr=false
```

---

## Information

`ng serve` и `ng build` — это две строчки в терминале, за которыми стоит один файл.
Пока не разберёшься с `angular.json`, любая нестандартная задача («а как собрать
для стенда?», «почему CI падает на размере?», «куда воткнуть глобальный SCSS?»)
превращается в спросить у нейронки, что куда должно быть.

В этой задаче мы намеренно ломаем сборку бюджетами, чиним её, добавляем своё окружение
и смотрим внутрь бандла.

---

## Statement

### Шаг 1. Инвентаризация

Не открывая редактор, ответьте через `ng config`:

```bash
ng config projects.task-board.architect.build.builder
ng config projects.task-board.architect.build.defaultConfiguration
ng config projects.task-board.prefix
```

> Если у вас проект на новом билдере, ключ может называться `targets`, а не `architect` —
> посмотрите, какой из них есть в вашем `angular.json`, и используйте его.

Затем найдите в файле и выпишите: где задан выходной каталог, где подключаются глобальные
стили, где перечислены бюджеты.

### Шаг 2. Дефолты схематик для команды

Зафиксируйте соглашения так, чтобы их не приходилось повторять флагами.
После этого шага `ng g c foo` должен **сам** создавать компонент со SCSS и `OnPush`.

Сделайте это **командами `ng config`**, не редактируя JSON руками:

- стиль компонентов — `scss`;
- стратегия обнаружения изменений — `OnPush`;
- спеки для сервисов — не генерировать.

Проверка:

```bash
ng g c smoke-test --dry-run
```

В выводе должны быть `.scss`, и в сгенерированном коде — `OnPush`.
(`--dry-run` ничего не создаст, но покажет список файлов; чтобы увидеть содержимое,
сгенерируйте компонент по-настоящему, посмотрите и удалите.)

### Шаг 3. Окружения

Добавьте файлы окружений и третье окружение — `staging`:

```bash
ng g environments
```

Нужно получить:

- `environment.ts` — прод: `production: true`, `apiUrl: 'https://api.example.com'`;
- `environment.development.ts` — дев: `production: false`, `apiUrl: 'http://localhost:3000'`;
- `environment.staging.ts` — стенд: `production: false`, `apiUrl: 'https://staging.example.com'`.

Выведите `apiUrl` в корневом компоненте, чтобы результат было видно в браузере.

### Шаг 4. Своя конфигурация сборки

Добавьте в таргет `build` конфигурацию `staging`, которая:

- подменяет `environment.ts` на `environment.staging.ts`;
- включает `outputHashing: "all"`;
- задаёт `baseHref: "/staging/"`;
- включает `sourceMap` (в отличие от прода).

И добавьте соответствующую конфигурацию в таргет `serve`.

Проверка:

```bash
ng build --configuration staging
ng serve --configuration staging --port 4301
```

В браузере должен быть `https://staging.example.com`, в `dist/` — файлы с хешами,
а в `index.html` — `<base href="/staging/">`.

### Шаг 5. Сломать сборку бюджетами и починить

Сначала уроните сборку намеренно:

1. В конфигурации `production` поставьте `initial` → `maximumError: "100kb"`.
2. Запустите `ng build`. Убедитесь, что сборка **падает**, и прочитайте текст ошибки:
   там указан фактический размер.
3. Верните адекватный лимит: `maximumWarning: "500kb"`, `maximumError: "1mb"`.

Затем добавьте бюджет, которого не было:

```jsonc
{ "type": "anyComponentStyle", "maximumWarning": "2kb", "maximumError": "4kb" }
```

Проверьте, что он работает: временно раздуйте стиль одного компонента
(например, сгенерируйте много CSS-правил) и убедитесь, что появилось предупреждение.

### Шаг 6. Заглянуть внутрь бандла

```bash
ng build --configuration production --stats-json
```

Найдите получившийся файл статистики в `dist/` и откройте его визуализатором:

```bash
# для application-билдера (esbuild)
npx esbuild-visualizer --metadata dist/task-board/stats.json --open

# универсальный вариант по sourcemap
npx source-map-explorer "dist/task-board/browser/*.js"
```

Ответьте:

1. Какая часть бандла приходится на код Angular, а какая — на ваш?
2. Что попало в `initial`?
3. Что изменится в размерах, если собрать с `--configuration development`? Почему?

### Шаг 7. Посмотреть на «взрослый» конфиг

Вернитесь в этот репозиторий и найдите в [angular.json](../../../../angular.json)
проект `angular-injection-token`. Ответьте:

1. Какой у него билдер и чем он отличается от билдера в вашем `task-board`?
2. Почему у этого проекта заполнено поле `root`, а у вашего оно пустое?
3. Как имя проекта в `angular.json` связано с командой
   `npm run serve:angular-injection-token` в [package.json](../../../../package.json)?
4. Что нужно поменять в конфиге проекта, чтобы перевести его на
   `@angular/build:application`? (Отвечать словами, менять ничего не нужно —
   миграция для этого называется `use-application-builder`.)

---

## Constraints

- Шаги 2 выполняется **только через `ng config`** — руками JSON в этом шаге не править.
- В шагах 4–5 править `angular.json` руками можно и нужно: это тоже навык.
- Не поднимайте лимиты бюджетов «чтобы собралось» — в шаге 5 нужно именно увидеть падение
  и осознанно вернуть разумные значения.
- Все изменения должны быть воспроизводимы: после `rm -rf dist && ng build -c staging`
  результат тот же.

---

## Чек-лист сдачи

- [ ] `ng g c foo` без флагов создаёт SCSS + `OnPush`
- [ ] Есть три файла окружений, `apiUrl` виден в UI и меняется в зависимости от конфигурации
- [ ] `ng build -c staging` собирает с хешами и `<base href="/staging/">`
- [ ] `ng serve -c staging` поднимается и показывает staging-адрес
- [ ] Вы видели своими глазами падение сборки по бюджету и текст ошибки
- [ ] Бюджет `anyComponentStyle` срабатывает на раздутом стиле
- [ ] Открыт визуализатор бандла, вы можете назвать три самых тяжёлых куска
- [ ] Есть ответы на вопросы шага 7

---

## Hint

<details>
  <summary>Подсказка 1 — синтаксис ng config</summary>

Путь — это точечная нотация по JSON, значение идёт вторым аргументом:

```bash
ng config schematics.@schematics/angular:component.style scss
```

Если ключа ещё нет, `ng config` создаст его. Прочитать значение — та же команда
без второго аргумента. Проверить весь блок:

```bash
ng config schematics
```

В некоторых оболочках `@` и `:` в пути стоит взять в кавычки.

</details>

<details>
  <summary>Подсказка 2 — как выглядит конфигурация staging</summary>

```jsonc
"configurations": {
  "production": { /* ... */ },
  "staging": {
    "baseHref": "/staging/",
    "outputHashing": "all",
    "sourceMap": true,
    "fileReplacements": [
      {
        "replace": "src/environments/environment.ts",
        "with": "src/environments/environment.staging.ts"
      }
    ]
  },
  "development": { /* ... */ }
}
```

Для `serve` конфигурация ссылается на сборку:

```jsonc
"serve": {
  "configurations": {
    "staging": { "buildTarget": "task-board:build:staging" }
  }
}
```

</details>

<details>
  <summary>Подсказка 3 — почему environment не подменился</summary>

Три типичные причины:

1. импорт идёт из `environment.staging`, а не из `environment` — подменяется только тот
   файл, который указан в `replace`, поэтому в коде всегда импортируется базовый;
2. конфигурация не указана: `ng build` без `-c staging` возьмёт `defaultConfiguration`;
3. пути в `fileReplacements` относительны корню workspace, а не `src`.

</details>

<details>
  <summary>Подсказка 4 — где искать stats.json</summary>

Имя и расположение зависят от билдера и `outputPath`. Проще найти:

```bash
find dist -name "stats.json"
```

Если файла нет — проверьте, что флаг `--stats-json` действительно поддерживается вашим
билдером, и что сборка не упала раньше.

</details>

---

## Что дальше

Если хочется добить тему до конца:

- прогоните `ng update --dry-run` в этом репозитории и посмотрите, что предлагается;
- выполните `ng add @angular/material` в `task-board` и сравните `git diff` с тем,
  что сделал бы обычный `npm install`;
- сгенерируйте библиотеку `ng g library ui-kit` и подключите её в приложение
  через path mapping.
