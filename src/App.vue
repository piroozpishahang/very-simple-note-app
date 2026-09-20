<script setup lang="ts">
import { ref, watch } from "vue";
import { open, confirm } from "@tauri-apps/plugin-dialog";
import {
  readDir,
  readTextFile,
  writeTextFile,
  remove,
  rename,
} from "@tauri-apps/plugin-fs";

const notesDirectory = ref<string | null>(null);
const notes = ref<string[]>([]);

const currentNote = ref<string | null>(null);
const content = ref("");

let saveTimer: ReturnType<typeof setTimeout> | null = null;

watch(content, () => {
  if (!currentNote.value || !notesDirectory.value) {
    return;
  }

  if (saveTimer) {
    clearTimeout(saveTimer);
  }

  saveTimer = setTimeout(() => {
    saveNote();
  }, 500);
});

async function chooseNotesDirectory() {
  const selected = await open({
    directory: true,
    multiple: false,
    title: "Choose Notes Directory",
  });

  if (typeof selected !== "string") {
    return;
  }

  notesDirectory.value = selected;
  await loadNotes(selected);
}

async function loadNotes(directory: string) {
  const entries = await readDir(directory);

  notes.value = entries
    .filter((entry) => entry.isFile && entry.name.endsWith(".md"))
    .map((entry) => entry.name);
}

async function openNote(note: string) {
  if (!notesDirectory.value) {
    return;
  }

  const path = `${notesDirectory.value}/${note}`;

  content.value = await readTextFile(path);
  currentNote.value = note;
}

async function saveNote() {
  if (!notesDirectory.value || !currentNote.value) {
    return;
  }

  const path = `${notesDirectory.value}/${currentNote.value}`;

  await writeTextFile(path, content.value);
}

async function deleteNote() {
  if (!notesDirectory.value || !currentNote.value) {
    return;
  }

  const confirmed = await confirm(
    `Are you sure you want to delete "${currentNote.value}"?`,
    {
      title: "Delete Note",
      kind: "warning",
    },
  );

  if (!confirmed) {
    return;
  }

  const path = `${notesDirectory.value}/${currentNote.value}`;

  try {
    await remove(path);

    const deletedNote = currentNote.value;

    currentNote.value = null;
    content.value = "";

    notes.value = notes.value.filter((note) => note !== deletedNote);
  } catch (error) {
    console.error("Delete failed:", error);
  }
}

async function renameNote() {
  if (!notesDirectory.value || !currentNote.value) {
    return;
  }

  const oldName = currentNote.value;

  const newName = window.prompt(
    "Enter the new note name:",
    oldName.replace(/\.md$/, ""),
  );

  if (newName === null) {
    return;
  }

  const trimmedName = newName.trim();

  if (!trimmedName) {
    return;
  }

  const filename = trimmedName.endsWith(".md")
    ? trimmedName
    : `${trimmedName}.md`;

  if (filename === oldName) {
    return;
  }

  if (notes.value.includes(filename)) {
    window.alert("A note with this name already exists.");
    return;
  }

  const oldPath = `${notesDirectory.value}/${oldName}`;
  const newPath = `${notesDirectory.value}/${filename}`;

  try {
    await rename(oldPath, newPath);

    const index = notes.value.indexOf(oldName);

    if (index !== -1) {
      notes.value[index] = filename;
    }

    currentNote.value = filename;
  } catch (error) {
    console.error("Rename failed:", error);
  }
}

async function createNote() {
  console.log("createNote called");

  if (!notesDirectory.value) {
    console.log("No notes directory");
    return;
  }

  console.log("Directory:", notesDirectory.value);

  const filename = "New Note.md";
  const path = `${notesDirectory.value}/${filename}`;

  console.log("Creating:", path);

  await writeTextFile(path, "");

  console.log("File created");

  await loadNotes(notesDirectory.value);

  currentNote.value = filename;
  content.value = "";
}
</script>

<template>
  <div class="app">
    <aside class="sidebar">
      <h1>Notes</h1>

      <button class="new-note" :disabled="!notesDirectory" @click="createNote">
        + New Note
      </button>

      <button class="choose-folder" @click="chooseNotesDirectory">
        Choose Notes Folder
      </button>

      <p v-if="notesDirectory" class="directory">
        {{ notesDirectory }}
      </p>

      <div class="notes">
        <button
          v-for="note in notes"
          :key="note"
          class="note-item"
          :class="{ active: currentNote === note }"
          @click="openNote(note)"
        >
          {{ note }}
        </button>
      </div>
    </aside>

    <main class="editor">
      <header class="editor-header">
        <span>
          {{ currentNote ?? "No note selected" }}
        </span>

        <div v-if="currentNote" class="editor-actions">
          <button @click="renameNote">Rename</button>

          <button @click="saveNote">Save</button>

          <button @click="deleteNote">Delete</button>
        </div>
      </header>

      <textarea
        v-model="content"
        :disabled="!currentNote"
        placeholder="Select a note..."
      />
    </main>
  </div>
</template>
