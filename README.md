# axios-interceptors
Using axios interceptors to refresh a JWT

I struggle a little trying to find out how to use axios interceptors to refresh an expired JWT token, and I did not want to change my current project structure to match other people's implementation, which are good, but would result in a real hassle to change. Then I came out with the below.

I use a global.ts file to store global API URL, you can use whatever fits better on your project.

Something like:

./static/global.ts
```typescript
const URL:any = {
  API: 'YOUR_API_URL',
  LOGIN: '/user/login',
  REFRESH_TOKEN: '/user/refresh,
  GET_DATA: '/data/'
}

export { URL }
```

Then I create an Axios request handler which I call my "api plugin", and use a temp cookie to store the JWT

./plugin/api.ts
```typescript
import axios from 'axios'
import Cookies from 'js-cookie'
import { URL } from '../static/global'

// Axios CRUD methods
const api = {
  async get(url:string) {
    let resp = await http.get(url)
    return resp
  },
  async getwithHeaders(url:string, options:any) {
    let resp = await axios
                    .get(url, options)
                    .catch((error:any) => {
                      this.errorHandler(error)                                    
                    })
    return resp
  },
  async post(url:string, data:any) {
    let resp = await axios
              .post(url, data)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async putwithHeaders (url:string, data:any, options:any) {
    let resp = await axios
              .put(url, data, options)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async patchwithHeaders (url:string, data:any, options:any) {
    let resp = await axios
              .patch(url, data, options)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async postwithHeaders(url:string, data:any, options:any) {
    let resp = await axios
              .post(url, data, options)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async deletewithHeaders(url:string, options:any) {
    let resp = await axios
                .delete(url, options)
                .catch((error:any) => {
                  this.errorHandler(error)
                })
  },
  errorHandler (error:any) {
    try {
      if (!error.response.data) {
        throw 'Api is offline'
      }

      throw 'Api error'
    } catch (e) {
      console.log (e)
    }
  }
}

export default api
```

Then whenever I need to make an api call, I just import the plugin to the Vue component and use any of the provided CRUD methods, usually, I would add the api calls to a store file which is also called itself into a Vue component.

Then, whenever you make a call to a protected api endpoint, the request should include a provided JWT and just include this in your headers. Lets, say I have a logint store like this

./store/login.ts

```typescript
import { defineStore } from 'pinia'
import api from '../plugins/api'
import { URL } from '../static/global'
import Cookies from 'js-cookie'

export const loginStore = defineStore({
  id: 'loginStore',
  state: () => ({
    loginError: false,
    logged: false,
    accessToken: '',
  }),
  actions: {    
    authLogin (data: any) {
      try {
        let resp = api.post(URL.LOGIN, data)
        resp
        .then((res:any) => {
          if (res?.data) {
            const token = res?.data.token
            const refreshToken = res?.data.refreshToken            
            this.setAuthCookie(token, refreshToken)
            this.logged = true
          }          
        })
        .catch ((error:any) => {
          this.loginError = true
        })
      } catch (error) {
        console.log(error)
      }
    },
    setAuthCookie (token:string, refToken:string) {
      Cookies.set('token', token, {expires: 2/12, secure: true, sameSite: 'strict'})
      Cookies.set('refToken', refToken, {expires: 2/12, secure: true, sameSite: 'strict'})
    }
  }
})

```

Where loginStore.authlogin is expecting to receive an object (data), which is just an JS object, like {'userName': 'yourUser', 'password': 'Pass1234'}, on a success match it should return an active token, but also a renewal token which you should use on the first one expiration.

Then any other API endpoint which is protected should be expecting to receive an authenticate this token, then lets say that every request you should add the corresponding header including the token

**TO BE CONTINUED**
