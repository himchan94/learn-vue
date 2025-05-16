<template>
  <div
    class="bg-white shadow-xl rounded-lg max-w-2xl mx-auto my-10 overflow-hidden">
    <!-- 헤더 섹션 -->
    <div class="bg-gradient-to-r from-green-500 to-green-600 px-8 py-7">
      <h1 class="text-2xl font-bold text-center text-white">Vue Todo 앱</h1>
    </div>

    <div class="p-8 md:p-10">
      <!-- 할일 추가 폼 -->
      <div class="flex mb-10">
        <input
          v-model="newTodoText"
          @keyup.enter="addTodo"
          placeholder="할일을 입력하세요"
          class="flex-1 border border-gray-300 p-4 rounded-l-md focus:outline-none focus:ring-2 focus:ring-green-500 focus:border-transparent shadow-sm" />
        <button
          @click="addTodo"
          class="bg-green-500 hover:bg-green-600 text-white px-6 py-4 rounded-r-md transition duration-200 shadow-sm">
          추가
        </button>
      </div>

      <!-- 할일 목록 -->
      <div class="flex flex-col gap-5">
        <div
          v-if="todos.length === 0"
          class="text-center py-12 text-gray-500 bg-gray-50 rounded-lg">
          할일이 없습니다. 새로운 할일을 추가해보세요!
        </div>

        <div
          v-for="(todo, index) in todos"
          :key="todo.id"
          class="flex gap-4 items-center p-5 bg-gray-50 rounded-md hover:bg-gray-100 transition-colors duration-200 shadow-sm"
          :class="{ 'bg-gray-100 opacity-75': todo.completed }">
          <!-- 완료 체크박스 -->
          <div class="flex items-center justify-center">
            <input
              type="checkbox"
              v-model="todo.completed"
              class="h-6 w-6 text-green-500 border-2 border-gray-300 rounded-md focus:ring-green-400 focus:ring-2 cursor-pointer" />
          </div>

          <!-- 할일 텍스트 (수정 모드가 아닐 때) -->
          <span
            v-if="!todo.editing"
            @dblclick="startEditing(index)"
            class="flex-1 mx-3 cursor-pointer text-gray-800 text-base"
            :class="{ 'line-through text-gray-400': todo.completed }">
            {{ todo.text }}
          </span>

          <!-- 할일 수정 입력 필드 (수정 모드일 때) -->
          <input
            v-else
            v-model="todo.text"
            @blur="finishEditing(index)"
            @keyup.enter="finishEditing(index)"
            @keyup.esc="cancelEditing(index)"
            v-focus
            class="flex-1 mx-3 p-3 border border-gray-300 rounded focus:outline-none focus:ring-2 focus:ring-green-500 shadow-sm" />

          <!-- 삭제 버튼 -->
          <button
            @click="removeTodo(index)"
            class="ml-4 px-4 py-2 bg-red-500 hover:bg-red-600 text-white rounded-md text-sm transition duration-200">
            삭제
          </button>
        </div>
      </div>

      <!-- 할일 통계 -->
      <div
        v-if="todos.length > 0"
        class="mt-10 flex justify-between items-center bg-gray-50 p-6 rounded-md text-sm shadow-sm">
        <div class="flex gap-6">
          <span class="text-gray-600 font-medium"
            >전체: {{ todos.length }}개</span
          >
          <span class="text-green-600 font-medium"
            >완료: {{ completedCount }}개</span
          >
          <span class="text-blue-600 font-medium"
            >미완료: {{ activeCount }}개</span
          >
        </div>
        <div class="flex gap-3">
          <button
            @click="clearCompleted"
            v-if="completedCount > 0"
            class="px-5 py-2.5 text-xs bg-gray-200 hover:bg-gray-300 rounded-md transition duration-200 font-medium">
            완료된 항목 삭제
          </button>
          <button
            @click="clearAll"
            class="px-5 py-2.5 text-xs bg-red-100 hover:bg-red-200 text-red-700 rounded-md transition duration-200 font-medium">
            모든 항목 삭제
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
// Vue의 Composition API를 사용합니다
import {
  ref,
  computed,
  watch,
  onMounted,
  nextTick,
  defineExpose,
  type Directive,
} from "vue";

// 커스텀 디렉티브 정의 - 자동 포커스를 위한 디렉티브
const vFocus: Directive = {
  mounted: (el: HTMLElement) => {
    // 입력 요소에 포커스
    if (el instanceof HTMLInputElement) {
      el.focus();
    }
  },
};

// Todo 타입 정의
interface Todo {
  id: number;
  text: string;
  completed: boolean;
  editing: boolean;
  prevText?: string;
}

// Todo 목록 상태 관리 (반응형 데이터)
const todos = ref<Todo[]>([]);
const newTodoText = ref("");

// 로컬 스토리지에서 데이터 불러오기
onMounted(() => {
  const savedTodos = localStorage.getItem("vue-todos");
  if (savedTodos) {
    todos.value = JSON.parse(savedTodos);
  } else {
    // 기본 Todo 항목 설정
    todos.value = [
      {
        id: Date.now(),
        text: "Vue 문법 학습하기",
        completed: false,
        editing: false,
      },
      {
        id: Date.now() + 1,
        text: "상태관리 (Pinia/Vuex) 알아보기",
        completed: false,
        editing: false,
      },
      {
        id: Date.now() + 2,
        text: "SSR과 Nuxt.js 살펴보기",
        completed: false,
        editing: false,
      },
    ];
    // 새로 생성한 기본 Todo 항목들을 로컬 스토리지에 저장
    localStorage.setItem("vue-todos", JSON.stringify(todos.value));
  }
});

// 데이터가 변경될 때마다 로컬 스토리지에 저장 (watch 사용)
watch(
  todos,
  (newTodos) => {
    localStorage.setItem("vue-todos", JSON.stringify(newTodos));
  },
  { deep: true }
);

// 새로운 Todo 추가 (이벤트 핸들링)
const addTodo = () => {
  const text = newTodoText.value.trim();
  if (text) {
    // 새 Todo 객체 생성
    todos.value.push({
      id: Date.now(),
      text,
      completed: false,
      editing: false,
    });
    // 입력 필드 초기화
    newTodoText.value = "";
  }
};

// Todo 삭제 (배열 변경)
const removeTodo = (index: number) => {
  todos.value.splice(index, 1);
};

// 수정 모드 시작
const startEditing = (index: number) => {
  // 이전 텍스트 저장 (취소 시 복원용)
  todos.value[index].prevText = todos.value[index].text;
  todos.value[index].editing = true;

  // DOM 업데이트 후 입력 필드에 포커스 (nextTick 사용)
  nextTick(() => {
    const inputElements = document.querySelectorAll(".edit-input");
    if (inputElements[0]) {
      (inputElements[0] as HTMLInputElement).focus();
    }
  });
};

// 수정 완료
const finishEditing = (index: number) => {
  const todo = todos.value[index];
  // 입력이 비어있으면 삭제
  if (todo.text.trim() === "") {
    removeTodo(index);
  } else {
    todo.editing = false;
    delete todo.prevText;
  }
};

// 수정 취소
const cancelEditing = (index: number) => {
  const todo = todos.value[index];
  // 이전 텍스트로 복원
  if (todo.prevText !== undefined) {
    todo.text = todo.prevText;
  }
  todo.editing = false;
  delete todo.prevText;
};

// 완료된 항목 모두 삭제
const clearCompleted = () => {
  todos.value = todos.value.filter((todo) => !todo.completed);
};

// 모든 항목 삭제
const clearAll = () => {
  todos.value = [];
};

// 계산된 속성 (computed)
const completedCount = computed(() => {
  return todos.value.filter((todo) => todo.completed).length;
});

const activeCount = computed(() => {
  return todos.value.length - completedCount.value;
});

// 컴포넌트 메서드 노출 (부모 컴포넌트에서 접근 가능)
defineExpose({
  clearAll: () => {
    todos.value = [];
  },
});
</script>

<style>
/* 직접 스타일링을 추가하거나 Tailwind 유틸리티 클래스를 사용합니다 */
.todo-app-custom {
  @apply bg-white shadow-lg rounded-lg p-8 w-full;
}

.todo-input {
  @apply flex-1 border border-gray-300 p-3 rounded-l-md focus:outline-none focus:ring-2 focus:ring-green-500 focus:border-transparent;
}

.add-button {
  @apply bg-green-500 hover:bg-green-600 text-white px-5 py-3 rounded-r-md transition duration-200;
}

.delete-button {
  @apply ml-3 px-3 py-1.5 bg-red-500 hover:bg-red-600 text-white rounded-md text-sm transition duration-200;
}
</style>
