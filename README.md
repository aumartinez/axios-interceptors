# axios-interceptors
Using axios interceptors to refresh a JWT

I struggle a little trying to find out how to use axios interceptors to refresh an expired JWT token, and I did not want to change my current project structure to match other people's implementation, which are good, but would result in a real hassle to change. Then I came out with the below.

I use a global.ts file to store global API URLs, but you can use whatever fits better on your project.

Something like:

./static/global.ts
```typescript
const URL:any = {
  API: 'YOUR_API_URL',
  LOGIN: '/user/login',
  REFRESH_TOKEN: '/user/refresh',
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
    let resp = await axios.get(url)
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

Then any other API endpoint which is protected should be expecting to receive an authenticate this token, then lets say that to every request you should add the corresponding header including the token.

For example in a different store component, we may have a function/action to call the data API and get the returned payload, notice the required token:

./store/data.ts
```typescript
getData(data:any) {
      try {
        let url = URL.GET_DATA
        let token = Cookies.get('token')
        let options = {
          headers: {
            'x-access-token': token
          }
        }
        let resp = api.getwithHeaders(url, options)

        resp
        .then (res => {
          console.log(res)
          // Do something with the returned data
        })
      } catch (error) {
        console.log(error)
      }
    }
```

Then this is where we meet the question, where do I refresh this token when it expires? We do have the auth token and the refresh token both available in local cookies. Auth token is used to make calls to the API but when it expires, it will throw an error and we need to use the refresh token to create a new auth token. All this without breaking the user flow in the app.

Here is where axios interceptors come, as it says they do "intercept" the request made through axios and you can listen to the endpoint responses on every API call, whenever the interceptor receive an invalid response, then we work on a callback to refresh the auth token.

Here is an example for the above implementation:
./plugins/api.ts
```typescript
...
// Axios interceptors
const http = axios.create()

http.interceptors.request.use (
  async config => {
    const token = Cookies.get('token') ? Cookies.get('token') : ''
    config.baseURL = URL.API    
    config.headers = {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      'x-access-token': token
    }
    return config
  },
  error => {
    Promise.reject(error)
  }
)

http.interceptors.response.use (
  (res) => { return res },
  async function (error) {
    const config = error?.config
    if (error.response.status === 401 && !config?.sent) {
      config.sent = true
      let resp = await refreshAccessToken()
      const token = resp?.data.token
      const refToken = resp?.data.refreshToken
      Cookies.set('token', token, {expires: 2/12, secure: true, sameSite: 'strict'})
      Cookies.set('refToken', refToken, {expires: 2/12, secure: true, sameSite: 'strict'})
      axios.defaults.headers.common['x-access-token'] = token ? token : ''      
      return http(config)
    }
    return Promise.reject(error)    
  }  
)

async function refreshAccessToken () {
  let refToken = Cookies.get('refToken') ? Cookies.get('refToken') : ''
  let url = URL.API + URL.REFRESH_TOKEN

  let data = {
    'refreshToken': refToken
  }

  let resp = await axios
                  .post(url, data)
                  .catch(error => {
                    console.log(error)
                  })
  return resp
}
...
```

Adding the code to call to interceptos before the axios CRUD handlers and since we want to the interceptors to handle axios calls, that's why we are creating a new instance of the axios class and need to replace our axios functions and change them for the new axios instance

```typescript
// Axios CRUD methods
const api = {
  async get(url:string) {
    let resp = await http.get(url)
    return resp
  },
  async getwithHeaders(url:string, options:any) {
    let resp = await http
                    .get(url, options)
                    .catch((error:any) => {
                      this.errorHandler(error)                                    
                    })
    return resp
  },
  async post(url:string, data:any) {
    let resp = await http
              .post(url, data)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async putwithHeaders (url:string, data:any, options:any) {
    let resp = await http
              .put(url, data, options)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async patchwithHeaders (url:string, data:any, options:any) {
    let resp = await http
              .patch(url, data, options)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async postwithHeaders(url:string, data:any, options:any) {
    let resp = await http
              .post(url, data, options)
              .catch((error:any) => {
                this.errorHandler(error)
              })
    return resp
  },
  async deletewithHeaders(url:string, options:any) {
    let resp = await http
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
