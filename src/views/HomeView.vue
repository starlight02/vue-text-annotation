<script setup>
import TheWelcome from '../components/TheWelcome.vue'
import { RouterLink, RouterView, useRouter } from 'vue-router'
import HelloWorld from '../components/HelloWorld.vue';
import { NButton } from 'naive-ui';

const router = useRouter();

const toJobPage = () => {
  router.push({name: 'Job'});
}
</script>

<template>
  <main>
    <header>
    <!-- <img alt="Vue logo" class="logo" src="@/assets/logo.svg" width="125" height="125" /> -->

      <div class="wrapper">
        <HelloWorld msg="You did it!" />
        <nav>
          <RouterLink to="/">Home</RouterLink>
          <RouterLink to="/about">About</RouterLink>
        </nav>
        <n-button @click="toJobPage">材料审批作业</n-button>
      </div>
    </header>
    <!-- <TheWelcome /> -->
  </main>
</template>
