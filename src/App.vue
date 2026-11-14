<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

interface Lesson {
  id: number
  subject: string
  location: string
  price: number
  spaces: number
  icon: string
  image: string
}

interface ApiLesson {
  id: number
  topic: string
  location: string
  price: number
  icon: string
  space: number
  image: string
  currency?: string
}

interface CartItem {
  lessonId: number
  lesson: Lesson
  quantity: number
}

const images = import.meta.glob('./images/*.webp', {
  eager: true,
  query: '?url',
  import: 'default'
}) as Record<string, string>

const lessons = ref<Lesson[]>([])

const searchQuery = ref('')
const matchingLessonIds = ref<number[] | null>(null)
const searchError = ref('')


const lessonError = ref('')

async function loadLessons() {
  try {
    const response = await fetch('/lessons')

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`)
    }

    const data: ApiLesson[] = await response.json()

    lessons.value = data.map(lesson => ({
      id: lesson.id,
      subject: lesson.topic,
      location: lesson.location,
      price: lesson.price,
      spaces: lesson.space,
      icon: lesson.icon,
      image: images[`./images/${lesson.image}`] ?? ''
    }))

    lessonError.value = ''
  } catch (error) {
    lessonError.value = 'Unable to load lessons'
    console.error('Lesson loading failed:', error)
  }
}

onMounted(loadLessons)

async function searchLessons() {
  const query = searchQuery.value.trim()

  if (!query) {
    matchingLessonIds.value = null
    return
  }

  try {
    const response = await fetch(
      `/lessons?q=${encodeURIComponent(query)}`
    )

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`)
    }

    const results: { id: number }[] = await response.json()

    matchingLessonIds.value = results.map(lesson => lesson.id)
    searchError.value = ''
  } catch (error) {
    searchError.value = 'Search failed. Please try again.'
    console.error(error)
  }
}



const sortOption = ref('subject-asc')
const cart = ref<CartItem[]>([])
const showCart = ref(false)
const checkoutMessage = ref('')
const customerName = ref('')
const customerPhone = ref('')

const sortedLessons = computed(() => {
  const sorted = [...lessons.value]
  const [attribute, direction] = sortOption.value.split('-')

  sorted.sort((a, b) => {
    let result = 0

    switch (attribute) {
      case 'subject':
        result = a.subject.localeCompare(b.subject)
        break
      case 'location':
        result = a.location.localeCompare(b.location)
        break
      case 'price':
        result = a.price - b.price
        break
      case 'spaces':
        result = a.spaces - b.spaces
        break
    }

    return direction === 'desc' ? -result : result
  })


const matchingIds = matchingLessonIds.value

return matchingIds === null
  ? sorted
  : sorted.filter(lesson => matchingIds.includes(lesson.id))

})

const cartCount = computed(() =>
  cart.value.reduce((total, item) => total + item.quantity, 0)
)

const cartItems = computed(() =>
  cart.value.map(item => ({
    ...item,
    subtotal: item.lesson.price * item.quantity
  }))
)

const cartTotal = computed(() =>
  cartItems.value.reduce((total, item) => total + item.subtotal, 0)
)

const validName = computed(() =>
  /^[A-Za-z]+(?: [A-Za-z]+)*$/.test(customerName.value.trim())
)

const validPhone = computed(() =>
  /^[0-9]+$/.test(customerPhone.value.trim())
)

const canCheckout = computed(() =>
  cartCount.value > 0 && validName.value && validPhone.value
)

function addToCart(lesson: Lesson) {
  if (lesson.spaces <= 0) return

  const existingItem = cart.value.find(
    item => item.lessonId === lesson.id
  )

  if (existingItem) {
    existingItem.quantity++
  } else {
    cart.value.push({
      lessonId: lesson.id,
      lesson,
      quantity: 1
    })
  }

  lesson.spaces--
  checkoutMessage.value = ''
}

function removeFromCart(lessonId: number) {
  const index = cart.value.findIndex(
    item => item.lessonId === lessonId
  )

  if (index === -1) return

  const item = cart.value[index]

  if (item) {
    item.lesson.spaces += item.quantity
    cart.value.splice(index, 1)
  }

  if (cart.value.length === 0) {
    showCart.value = false
  }
}

function emptyCart() {
  cart.value.forEach(item => {
    item.lesson.spaces += item.quantity
  })

  cart.value = []
  showCart.value = false
}

function checkout() {
  if (!canCheckout.value) return

  const total = cartTotal.value
  const count = cartCount.value

  cart.value = []
  showCart.value = false

  checkoutMessage.value =
    `Demo order confirmed for ${customerName.value.trim()}. ` +
    `${count} lesson(s), total £${total.toFixed(2)}. ` +
    `No order has been sent to the server yet.`

  customerName.value = ''
  customerPhone.value = ''
}
</script>

<style src="./styles/style.css"></style>

<template>
  <div class="page">
  <p v-if="lessonError" class="error-message">
    {{  lessonError }}
  </p>
    <header class="header">
      <div>
        <p class="eyebrow">LEARN • CREATE • GROW</p>
        <h1>After School Lessons</h1>
        <p class="subtitle">
          Explore exciting after-school activities and
          discover something new.
        </p>
      </div>

      <button
        class="view-cart-button"
        :disabled="cartCount === 0 && !showCart"
        @click="showCart = !showCart"
      >
        🛒 {{ showCart ? 'View Lessons' : 'View Cart' }}
        <span class="cart-badge">{{ cartCount }}</span>
      </button>
    </header>

    <main>
      <p
        v-if="checkoutMessage"
        class="success-message"
        role="status"
      >
        {{ checkoutMessage }}
      </p>

      <section v-if="showCart" class="cart-panel">
        <div class="cart-heading">
          <div>
            <h2>Your Shopping Cart</h2>
            <p>{{ cartCount }} lesson(s) selected</p>
          </div>

          <button
            class="secondary-button"
            @click="showCart = false"
          >
            Continue Shopping
          </button>
        </div>

        <article
          v-for="item in cartItems"
          :key="item.lessonId"
          class="cart-item"
        >
          <img
            v-if="item.lesson.image"
            :src="item.lesson.image"
            :alt="item.lesson.subject"
            class="cart-image"
          />

          <div v-else class="cart-image placeholder">
            {{ item.lesson.icon }}
          </div>

          <div class="cart-item-info">
            <h3>{{ item.lesson.subject }}</h3>
            <p>{{ item.lesson.location }}</p>
            <p>
              £{{ item.lesson.price.toFixed(2) }}
              × {{ item.quantity }}
            </p>
          </div>

          <div class="cart-item-end">
            <strong>
              £{{ item.subtotal.toFixed(2) }}
            </strong>

            <button
              class="delete-button"
              @click="removeFromCart(item.lessonId)"
            >
              Remove
            </button>
          </div>
        </article>

        <div class="cart-total">
          <span>Total</span>
          <strong>£{{ cartTotal.toFixed(2) }}</strong>
        </div>

        <div class="checkout-form">
          <h2>Checkout Details</h2>

          <label for="customer-name">
            Customer Name
          </label>

          <input
            id="customer-name"
            v-model="customerName"
            type="text"
            placeholder="Enter your full name"
            autocomplete="name"
          />

          <p
            v-if="customerName && !validName"
            class="error-message"
          >
            Name must contain letters and spaces only.
          </p>

          <label for="customer-phone">
            Phone Number
          </label>

          <input
            id="customer-phone"
            v-model="customerPhone"
            type="tel"
            placeholder="Enter your phone number"
            autocomplete="tel"
          />

          <p
            v-if="customerPhone && !validPhone"
            class="error-message"
          >
            Phone number must contain numbers only.
          </p>

          <div class="cart-actions">
            <button
              class="empty-button"
              @click="emptyCart"
            >
              Empty Cart
            </button>

            <button
              class="checkout-button"
              :disabled="!canCheckout"
              @click="checkout"
            >
              Checkout · £{{ cartTotal.toFixed(2) }}
            </button>
          </div>
        </div>
      </section>

      <template v-else>
        <section class="toolbar">

<div class="lesson-search">
  <input
    v-model="searchQuery"
    type="search"
    placeholder="Search lessons, locations, prices..."
    @keyup.enter="searchLessons"
  />

  <button type="button" @click="searchLessons">
    Search
  </button>

  <button
    type="button"
    @click="searchQuery = ''; matchingLessonIds = null"
  >
    Clear
  </button>
</div>

          <div>
            <h2>Available Lessons</h2>
            <p>Choose a lesson to get started.</p>
          </div>

          <label for="sort">
            Sort by:
            <select id="sort" v-model="sortOption">
              <option value="subject-asc">
                Subject (A–Z)
              </option>
              <option value="subject-desc">
                Subject (Z–A)
              </option>
              <option value="location-asc">
                Location (A–Z)
              </option>
              <option value="location-desc">
                Location (Z–A)
              </option>
              <option value="price-asc">
                Price (Low to High)
              </option>
              <option value="price-desc">
                Price (High to Low)
              </option>
              <option value="spaces-asc">
                Spaces (Low to High)
              </option>
              <option value="spaces-desc">
                Spaces (High to Low)
              </option>
            </select>
          </label>
        </section>

        <section class="lesson-grid">
          <article
            v-for="lesson in sortedLessons"
            :key="lesson.id"
            class="lesson-card"
          >
            <img
              v-if="lesson.image"
              :src="lesson.image"
              :alt="lesson.subject"
              class="lesson-image"
            />

            <div v-else class="lesson-image placeholder">
              {{ lesson.icon }}
            </div>

            <h3>{{ lesson.subject }}</h3>

            <p class="location">
              📍 {{ lesson.location }}
            </p>

            <div class="lesson-details">
              <p class="price">
                £{{ lesson.price.toFixed(2) }}
              </p>

              <p
                class="spaces"
                :class="{ 'low-spaces': lesson.spaces === 0 }"
              >
                {{ lesson.spaces }} spaces left
              </p>
            </div>

            <button
              class="add-button"
              :disabled="lesson.spaces === 0"
              @click="addToCart(lesson)"
            >
              {{ lesson.spaces > 0 ? 'Add to Cart' : 'Sold Out' }}
            </button>
          </article>
        </section>
      </template>
    </main>

    <footer>
      After School Lessons · Learn Something New
    </footer>
  </div>
</template>

