<template>
  <div class="p-4 bg-white max-w-md text-gray-700 shadow-md rounded-md mx-auto h-auto">
    <div v-if="fetchingTask" class="text-center">
      <p class="py-4 text-base text-center text-gray-500 font-medium">
        <ArrowPathIcon class="h-6 w-6 inline mr-2 animate-spin"></ArrowPathIcon> Fetching Task...
      </p>
    </div>
    <div v-else v-for="data in selectedTask" :key="data._id">
      <div v-if="updatingTask">
        <p class="py-4 text-base text-center text-gray-500 font-medium">
          <ArrowPathIcon class="h-6 w-6 inline mr-2 animate-spin"></ArrowPathIcon> Updating Task...
        </p>
      </div>
      <div v-else>
        <div class="mt-4">
          <label class="font-medium">Title</label>
          <input type="text" v-model="data.title" @input="title = $event.target.value" readonly="true"
            class="block mt-1 w-full rounded-md border-0 py-1.5 text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6 mb-1" />
        </div>
        <div class="mt-4">
          <label class="font-medium">Workspace</label>
          <input type="text" readonly="true" :value="data.workspace.title"
            class="block mt-1 w-full rounded-md border-0 py-1.5 text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6 mb-1" />
        </div>
        <div class="mt-4">
          <p class="font-medium">Todos</p>
          <div class="py-2 px-4">
            <div v-for="todo in data.todos" :key="todo.key">
              <input v-if="isReadOnly" type="checkbox"
                class="h-4 w-4 rounded border-gray-300 text-indigo-600 focus:ring-indigo-600" :checked="todo.done"
                disabled />
              <input v-else type="checkbox" v-model="todo.done" @input="todos = data.todos"
                class="h-4 w-4 rounded border-gray-300 text-indigo-600 focus:ring-indigo-600" :checked="todo.done" />
              <label class="ml-2">{{ todo.title }}</label>
            </div>
          </div>
        </div>
        <div class="my-4">
          <p class="font-medium mb-2">Comments</p>
          <div v-if="isComment" class="flex mb-4 gap-x-2">
            <input v-model="newComment" type="text"
              class="inline-block w-full rounded-md border-0 py-1.5 text-gray-900 shadow-sm ring-1 ring-inset ring-gray-300 placeholder:text-gray-400 focus:ring-2 focus:ring-inset focus:ring-indigo-600 sm:text-sm sm:leading-6"
              placeholder="Comment" />
            <button @click="addNewComment" class='px-2 py-1 bg-blue-500 text-white rounded-md'>comment</button>
          </div>
          <div v-if="allComments.length > 0" class="py-2 px-4 rounded-md bg-gray-200">
            <p class="text-sm" v-for="comment in allComments" :key="'comment.id'"> <span class="font-bold">{{ comment.name
            }}</span>: {{ comment.msg }}</p>
          </div>
        </div>
        <div v-if='!isExpired' class="flex justify-end gap-x-2">
          <button v-if='!isComment'
            class="px-3 py-1 ml-2 border border-indigo-500 text-indigo-500 rounded-md hover:bg-indigo-500 hover:text-white"
            @click="isComment = !isComment">comment</button>
          <div v-else class="flex">
            <button class="px-3 py-1 ml-2 border border-red-500 text-red-500 rounded-md hover:bg-red-500 hover:text-white"
              @click="isComment = !isComment">cancel comment</button>
          </div>
          <button v-if="isReadOnly"
            class="px-3 py-1 border border-indigo-500 text-indigo-500 rounded-md hover:bg-indigo-500 hover:text-white"
            @click="editTask">
            <PencilSquareIcon class="inline h-4 w-4"></PencilSquareIcon> Edit
          </button>
          <div v-else>
            <button class="px-3 py-1 border border-red-500 text-red-500 rounded-md hover:bg-red-500 hover:text-white"
              @click="cancelTask"> Cancel</button>
            <button class="px-3 py-1 ml-2 bg-indigo-500  text-white rounded-md hover:bg-indigo-600 hover:text-white"
              @click="updateTask"> Update </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { PencilSquareIcon, ArrowPathIcon } from '@heroicons/vue/24/outline'
import axios from 'axios'
import { useUserStore } from '@/stores/user'
import { useToast } from 'vue-toast-notification'

const $toast = useToast()
const router = useRouter()
const route = useRoute()
const user = useUserStore()

onBeforeMount(async () => {
  await axios.post('api/user/get-superadmins', {
    company: route.params.company
  })
    .then(res => {
      if (res.data.length == 0) {
        $toast.error(`No company named ${route.params.company} exist!`, { position: 'top-right' })
        router.back()
      }
    })
    .catch(err => {
      console.log(err)
    })
  if (user.role == 'admin' || user.role == 'superadmin') {
    $toast.info('User redirected as admin!', { position: 'top-right' })
    router.push(`/${route.params.company}/admin/task`)
  } else {
    await axios.post('api/auth/get-session', {
      userId: user.token
    })
      .then(res => {
        let auth = res.data.authenticated
        let userData = res.data.user
        let token = res.data.token
        if (auth) {
          user.setUser({ userData, auth, token })
          console.log('auth')
        }
      })
      .catch(err => {
        user.$reset()
        router.push(`/login`)
        $toast.error('User not authenticated!', { position: 'top-right' })
        console.log('not auth')
      })
  }
})

const fetchingTask = ref(true)

const title = ref('')
const workspace = ref('')
const taskId = ref('')
const todos = ref()

const selectedTask = ref()

const isReadOnly = ref(true)
const updatingTask = ref(false)

let isExpired = ref()

let allComments = ref()

onMounted(async () => {
  let newTask = await axios.post("api/task/selected-task", {
    id: route.params.id
  })
    .then(res => {
      selectedTask.value = res.data
      title.value = selectedTask.value[0].title
      workspace.value = selectedTask.value[0].workspace.id
      taskId.value = selectedTask.value[0]._id

      let thisdate = new Date()
      let dueDate = new Date(`${res.data[0].dueDate}`)

      if (thisdate > dueDate) {
        $toast.error('due date is expired', { position: 'top-right' })
        isExpired.value = true
        console.log('expired')
      } else {
        console.log('valid')
        isExpired.value = false
        console.log(dueDate)
      }

    })
    .catch(err => {
      $toast.error('Task doesnt exist', { position: 'top-right' })
      router.push(`/${route.params.company}/task`)
      console.log(err)
    })

  let theComment = await axios.post('/api/task/user/get-comment-task', {
    id: route.params.id
  })
    .then(res => {
      allComments.value = res.data.comment.reverse()
      console.log(res)
    })
    .catch(err => {
      console.log(err)
    })

  fetchingTask.value = false
})

const editTask = () => isReadOnly.value = !isReadOnly.value

const cancelTask = () => {
  isReadOnly.value = !isReadOnly.value
  router.go()
}

const updateTask = async () => {
  updatingTask.value = true
  await axios.post('api/task/update-task', {
    taskId: taskId.value,
    title: title.value,
    workspaceId: workspace.value,
    todos: todos.value
  })
    .then(res => {
      $toast.info('Task updated', { position: 'top-right' })
      //router.go()
    })
    .catch(err => {
      $toast.error('Error occured', { position: 'top-right' })
      console.log(err)
    })

  setTimeout(() => {
    updatingTask.value = false
    isReadOnly.value = true
  }, 2000)
}

let isComment = ref(false)
let newComment = ref()
let newComments = ref([])

const addNewComment = async () => {
  let comment = {
    name: 'user',
    msg: newComment.value
  }
  await newComments.value.push(comment)
  newComment.value = await ''

  await axios.post('/api/task/user/comment-task', {
    comments: comment,
    id: route.params.id
  })
    .then(res => {
      router.go()
      console.log(res)
    })
    .catch(err => {
      console.log(err)
    })
}

const setIsComment = () => isComment.value != isComment.value
</script>

<route lang="yaml">
meta:
  layout: users
</route>