---
title: Загружаем большие файлы на сервер с Go и JS
description: 
keywords: 
---

# Загружаем большие файлы на сервер с Go и JS

## общий принцип

- разбиваем на чанки
- генерим чексуммы
- валидируем чанки
- загружаем
- сообщаем о завершении загрузки
- склеиваем файл
- удаляем временные файлы

## фронт

### 

### Разбиваем файл на чанки

```js
/**
 * @param {Blob} blob
 * @returns {Promise<string>}
 */
async function checksumBlob(blob) {
	const arrayBuffer = await blob.arrayBuffer()
	const hashBuffer = await crypto.subtle.digest("SHA-256", arrayBuffer)
	const hashArray = Array.from(new Uint8Array(hashBuffer))
	return hashArray.map(b => b.toString(16).padStart(2, "0")).join("")
}

const CHUNK_SIZE = 2 * 1024 * 1024

/** @typedef {{ chunk: Blob, checksum: string, index: number }} Chunk */

/**
 * @param {File} file
 * @returns {Promise<Chunk[]>}
 */
const splitChunks = async (file) => {
	const totalChunks = Math.ceil(file.size / CHUNK_SIZE)

	const chunks = []

	for (let i = 0; i < totalChunks; i++) {
		const start = i * CHUNK_SIZE
		const end = Math.min(file.size, start + CHUNK_SIZE)

		const chunk = file.slice(start, end)

		const checksum = await checksumBlob(chunk)

		chunks.push({ index: i, chunk, checksum })
	}

	return chunks
}
```

## бэк

## Заключение
