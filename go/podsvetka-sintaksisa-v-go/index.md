---
title: Подсветка синтаксиса кода в Go
description: Добавляем в блог подстветку синтаксиса
keywords: syntax, syntax highlighting, go, golang, ssr, ssg, блог, веб, веб-разработка, SEO
---

# Подсветка синтаксиса в Go

Мне мягко намекнули, что читать код без подсветки синтаксиса довольно трудно. А я не готов был мириться с тем, что вы начнёте уходить с сайта, поэтому решил эту самую подсветку реализовать.

Как и в случае с markdown, парсить и красить всё руками я не собирался, поэтому использовал готовую библиотеку [chroma](https://github.com/alecthomas/chroma). Но так как у меня уже есть markdown, да ещё и свой велосипед поверх него, пришлось немного поплясать с бубном.

Порядок действий такой:
- Находим все блоки кода в нашем контенте и вырезаем их в отдельный массив.
- Вместо блока кода в тексте оставляем идентификатор вида `CODE_BLOCK_N`.
- Отдельно форматируем код.
- Отдельно форматируем markdown.
- Объединяем.

Это разделение сделано для того, чтобы ни один из парсеров не сработал где не нужно.

## Парсим код

У меня получилась вот такая функция:

```go
type CodeBlock struct {
	Code string
	Lang string
	Id   string
}

func ParseCodeBlocks(md []byte) ([]CodeBlock, []byte) {
	lines := strings.Split(string(md), "\n")

	newLines := make([]string, 0, len(lines))

	var blockLines []string

	blocks := make([]CodeBlock, 0, len(lines))

	lang := ""

	for i, line := range lines {
		if blockLines != nil {
			if line == "```" {
				id := fmt.Sprintf("CODE_BLOCK_%d", i)

				blocks = append(blocks, CodeBlock{
					Lang: lang,
					Id:   id,
					Code: strings.Join(blockLines, "\n"),
				})

				newLines = append(newLines, id)

				blockLines = nil
				continue
			}
			blockLines = append(blockLines, line)
			continue
		}

		if len(line) > 2 && line[0:3] == "```" {
			lang = strings.Replace(line, "```", "", 1)
			blockLines = make([]string, 0, len(lines))
			continue
		}

		newLines = append(newLines, line)
	}

	return blocks, []byte(strings.Join(newLines, "\n"))
}
```
Если говорить человеческим языком, то происходит следующее:
- Разбиваем текст на отдельные строки и циклом пробегаемся по ним.
- Если встречаем строку, которая начинаеся с трёх символов `, то это начало блока кода. Для корректной подсветки на этой же строке добавляется название языка.
- Все строки после начала блока кода мы добавляем в отдельный массив.
- Когда встречаем вторую строку с тремя символами `, мы понимаем, что код кончился.
- Вместо блока мы добавляем заглушку с номером.
- Все остальные строки вставляем как обычно.

На входе:

```md
Это просто какой-то текст.

\`\`\`go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
}
\`\`\`
```

На выходе:

```md
Это просто какой-то текст.

CODE_BLOCK_1
```

Ну и функция возвращает два значения: новый текст и отдельно сниппеты кода.

## Форматируем код на Go

Как я уже сказал, мы используем готовую библиотеку. И у неё даже есть возможность сделать всё вызовом одной функции. Жаль только, что функция генерирует целую страницу да ещё и с inline стилями — нам это не подходит, поэтому придётся настраивать руками:

```go
func FormatCode(block CodeBlock) ([]byte, error) {
	lexer := lexers.Get(block.Lang)

	if lexer == nil {
		lexer = lexers.Fallback
	}

	lexer = chroma.Coalesce(lexer)

	style := styles.Get("monokai")

	iterator, err := lexer.Tokenise(nil, block.Code)

	if err != nil {
		return []byte{}, err
	}

	formatter := html.New(
		html.Standalone(false),
		html.WithClasses(true),
		html.WithLineNumbers(true),
	)

	var buf bytes.Buffer

	err = formatter.Format(&buf, style, iterator)

	return buf.Bytes(), err
}

func RenderCode(h []byte, blocks []CodeBlock) []byte {
	content := string(h)
	for _, v := range blocks {
		code, err := FormatCode(v)

		if err != nil {
			fmt.Println(err.Error())
			continue
		}

		content = strings.Replace(content, v.Id, string(code), 1)
	}

	return []byte(content)
}
```

## Компануем страницу

Всё, что нам осталось сделать, это отформатировать markdown и вставить наши блоки.

```go
func MdToHTML(input []byte) []byte {
	codeBlocks, md := ParseCodeBlocks(input)

	// create markdown parser with extensions
	extensions := parser.CommonExtensions | parser.AutoHeadingIDs | parser.NoEmptyLineBeforeBlock
	p := parser.NewWithExtensions(extensions)
	doc := p.Parse(md)

	// create HTML renderer with extensions
	htmlFlags := mdHtml.CommonFlags | mdHtml.HrefTargetBlank | mdHtml.LazyLoadImages | mdHtml.NofollowLinks | mdHtml.NoreferrerLinks | mdHtml.NoopenerLinks
	opts := mdHtml.RendererOptions{Flags: htmlFlags}
	renderer := mdHtml.NewRenderer(opts)

	h := markdown.Render(doc, renderer)

	return RenderCode(h, codeBlocks)
}
```

Примеры подсветки вы видите прямо на этой странице. Главное, не забыть подключить отдельный файл со стилями.

## Заключение

В целом, я доволен, что код теперь более читаемый. Надеюсь, вам это тоже удобно. Однако без компромисса не обошлось — итоговой размер страниц с кодом увеличился в два раза и теперь в среднем составляет `35kb`. А вы говорите, что покраска буков ничего не стоит.

Также в 10 раз увеличилось время сборки статики — с `5мс` до `50мс`. Пока что я не вижу в этом проблемы, но скоро мне придётся думать об оптимизации.
