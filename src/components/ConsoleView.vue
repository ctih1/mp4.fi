<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'

interface Message {
  style: 'error' | 'text' | 'command-run'
  text: string
  path: string
}

const lines = ref<Message[]>([])
const currentLine = ref('')
const input = ref<HTMLTextAreaElement | undefined>(undefined)

onMounted(() => {
  window.addEventListener('click', focusInput)
})

function focusInput() {
  input.value?.focus()
}

const commands = {
  help: { description: 'see list of commands', args: '<command>' },
  ls: { description: 'view contents of file', args: '' },
  pwd: { description: 'show current directory', args: '' },
  cat: { description: 'show contents of file', args: '<file path>' },
  cd: { description: 'change directory', args: '<directory>' },
  clear: { description: 'clear the screen', args: '' },
}

const fsData = {
  '/': {
    home: {
      ctih1: {
        'README.md':
          'This is just a stupid little project because I dont know what to do with this domain',
        'github.desktop': `
        [Desktop Entry]
        Encoding=UTF-8
        Name=This projects GitHub
        Type=Link
        URL=https://github.com/ctih1/mp4.fi
        `,
        'PURCHASE.md': 'If you want to buy this domain, email me at contact@mp4.fi',
        projects: {
          'frii.site': {
            'README.md': `
            By far my biggest project to date. Basically a free subdomain registrar.
            The first version was made in ~february 2024, but the login and signup system was forked from another one of my projects.
            Originally everything was saved into a single users.json file, but since I only had a few users, it didn't really matter.
            In March 2024 I decided to switch to mongodb, where the project resides to this day.
            BACKEND:
                The backend uses FastAPI, MongoDB, and a self-hosted instance of PowerDNS (migrated from Cloudflare in early 2025). The entirety of the backend is fully type-hinted, which makes it easy to work with

                Timeline:
                - first commit (march 1st 2024) - https://github.com/ctih1/frii.site-backend/commit/e51c74360fd3ce0ee6b9671af8a2c5ff1a24c330
                - first refactor (july 16th 2024) - https://github.com/ctih1/frii.site-backend/commit/e29c81549b3501a29347820ca982b2eb746d9f50
                - latest refactor (may 1st 2025) - https://github.com/ctih1/frii.site-backend/commit/dd6f2f132ec3bd739748a45b886db5783fb7d548
            FRONTEND:
                The first version of frii.site was written in pure HTML, CSS and JS. No tailwind, no jquery, nothing extra. It was pretty bad.

                Timeline:
                  - first rewrite and redesign in Svelte (may-june 2024) (https://web.archive.org/web/20240629145558/https://www.frii.site/)
                  - redesign for dark theme (october 2024) (originally from halloween theme)
                  - Shadcn rewrite (in progress as of july 2025, June 2025-)
          `,
          },
        },
      },
    },
  },
}

function convertToMap(obj: any): Map<string, any> {
  const map = new Map()
  for (const [key, value] of Object.entries(obj)) {
    map.set(
      key,
      typeof value === 'object' && value !== null && !Array.isArray(value)
        ? convertToMap(value)
        : value,
    )
  }
  return map
}

const fs = convertToMap(fsData)

let currentDirString = '/'
let currentDirectory = fs.get('/')!

function formatCommands() {
  return Object.entries(commands)
    .map(([cmd, { args, description }]) => `${cmd} ${args} - ${description}`)
    .join('\n')
}

const introMessage = `
mp4.fi
This is just a fun little project. Available commands:

${formatCommands()}
`

function print(style: 'error' | 'text' | 'command-run', message: string) {
  if (message.includes('\n')) {
    message.split('\n').forEach((line) => {
      lines.value.push({ style, text: line, path: currentDirString })
    })
  } else {
    lines.value.push({ style, text: message, path: currentDirString })
  }
}

function printError(message: string) {
  print('error', message)
}

function resolvePath(currentPath: string, inputPath: string): string {
  if (inputPath.startsWith('/')) {
    return inputPath
  }

  const currentParts = currentPath.split('/').filter(Boolean)
  const inputParts = inputPath.split('/').filter(Boolean)

  for (const part of inputParts) {
    if (part === '.') {
      continue
    } else if (part === '..') {
      currentParts.pop()
    } else {
      currentParts.push(part)
    }
  }

  return '/' + currentParts.join('/')
}

function runCommand(input: string) {
  const command = input.trim().split(' ')[0]
  const args = input.trim().split(' ').slice(1)

  switch (command) {
    case 'ls': {
      const path = args[0] ?? currentDirString
      const dir = getDirectory(path)
      if (!dir) {
        printError('No such directory')
        break
      }
      print('text', [...dir.keys()].join(' '))
      break
    }
    case 'cd': {
      if (!args[0]) {
        printError('No path provided')
        break
      }
      const newPath = resolvePath(currentDirString, args[0])
      navigateToFolder(newPath)
      break
    }
    case 'pwd':
      print('text', currentDirString)
      break

    case 'clear':
      lines.value = []
      break

    case 'cat': {
      const path = args[0]
      if (!path) {
        printError('No path given')
        return
      }
      getFileContents(path)
      break
    }

    case 'help':
      print('text', formatCommands())
      break

    default:
      printError(`${command}: command not found`)
  }
}

function getDirectory(path: string): any {
  if (!path || path === '') path = '/'
  const parts = path.split('/').filter(Boolean)
  let dir = fs.get('/')!
  for (const part of parts) {
    if (!dir.has(part)) {
      return null
    }
    dir = dir.get(part)
  }
  return dir
}

function getFileContents(path: string) {
  if (!path.startsWith('/')) {
    path = currentDirString.endsWith('/') ? currentDirString + path : currentDirString + '/' + path
  }

  const parts = path.split('/').filter(Boolean)
  const filename = parts.slice(-1)[0]
  const dirPath = parts.slice(0, -1).join('/')
  const dir = getDirectory('/' + dirPath)

  if (!dir || !filename || !dir.has(filename)) {
    printError('No such file or directory')
    return
  }

  const content = dir.get(filename)
  print('text', content)
}

function navigateToFolder(path: string) {
  if (path === '') path = '/'
  const dir = getDirectory(path)
  if (!dir) {
    printError('No such directory')
    return
  }
  if (!(dir instanceof Map)) {
    printError('Not a directory')
    return
  }
  currentDirectory = dir
  currentDirString = path
}

watch(currentLine, () => {
  if (currentLine.value.includes('\n')) {
    const value = currentLine.value.trim()
    currentLine.value = ''
    if (value.length > 0) {
      print('command-run', value)
      runCommand(value)
    }
  }
})

navigateToFolder('/home/ctih1')

print('text', introMessage)
</script>

<template>
  <div class="w-screen h-screen bg-black text-white p-4 font-mono flex flex-col">
    <div class="flex flex-col-reverse space-y-1 overflow-y-auto h-[calc(100%-2.5rem)]">
      <div v-for="(line, index) in lines.slice().reverse()" :key="index">
        <span v-if="line.style === 'error'" class="text-red-500">{{ line.text }}</span>
        <span v-else-if="line.style === 'command-run'"
          ><span class="text-blue-700">{{ line.path }}</span
          >$ {{ line.text }}
        </span>
        <span v-else>{{ line.text }}</span>
      </div>
    </div>

    <div class="flex items-start mt-2">
      <span class="mr-2"
        ><span class="text-blue-700">{{ currentDirString }}</span
        >$</span
      >
      <textarea
        ref="input"
        v-model="currentLine"
        class="bg-transparent outline-none resize-none w-full"
        rows="1"
        autofocus
        autocomplete="off"
        spellcheck="false"
      />
    </div>
  </div>
</template>
