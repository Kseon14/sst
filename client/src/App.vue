<template>
  <div id="app">
    <h2>Efir</h2>
    <div id="nav">
      <router-link to="/">shift</router-link>
      | <router-link to="/worker">worker</router-link>
<!--      | <router-link to="/admin">admin</router-link>-->
    </div>
    <router-view/>
  </div>
</template>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
#nav {
  padding: 10px;
}

#nav a {
  font-weight: bold;
  color: #2c3e50;
}

#nav a.router-link-exact-active {
  color: #42b983;
}

</style>
