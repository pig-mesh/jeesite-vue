<template>
<div/>
</template>

<script>
import {getQueryObject} from '/@/utils'
import { useUserStore } from '/@/store/modules/user';
import {useRouter} from "vue-router";

export default {
  name: 'sso',
  data() {
    return {
      loginForm: {
        ssoCode: "",
      }
    };
  },
  created () {
    const url = window.location.href
    let queryObject = getQueryObject(url)
    this.ssoCode = queryObject.code

    const userStore = useUserStore();
    const userRouter = useRouter()

    userStore.ssoLogin(this.ssoCode).then(res => {
      userRouter.push("/")
    });
  }
}
</script>
