<script setup>
import HelloWorld from './components/HelloWorld.vue'
</script>

<template>
  <div class="app-container">
    <div class="header">
      <h1 class="main-title">🍄 Bienvenue dans le monde de Mario</h1>
    </div>
    <HelloWorld msg="Champignon Clicker" />
  </div>
</template>

<style scoped>
.app-container {
  min-height: 100vh;
  background: linear-gradient(135deg, #4CAF50 0%, #8BC34A 50%, #CDDC39 100%);
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

.header {
  text-align: center;
  margin-bottom: 2rem;
  padding: 2rem;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 20px;
  backdrop-filter: blur(10px);
  border: 2px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

.main-title {
  font-size: 3rem;
  color: #fff;
  text-shadow: 3px 3px 0px #ff6b35, 6px 6px 0px #ff8e53;
  margin: 0;
  font-weight: bold;
  animation: bounce 2s infinite;
}


@keyframes bounce {
  0%, 20%, 50%, 80%, 100% {
    transform: translateY(0);
  }
  40% {
    transform: translateY(-10px);
  }
  60% {
    transform: translateY(-5px);
  }
}


/* Responsive */
@media (max-width: 768px) {
  .main-title {
    font-size: 2rem;
    text-shadow: 2px 2px 0px #ff6b35, 4px 4px 0px #ff8e53;
  }
  
  
  .header {
    padding: 1rem;
    margin-bottom: 1rem;
  }
}
</style>
