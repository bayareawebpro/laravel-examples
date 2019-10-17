# Wrapper with Laravel / Symfony Conditionals

```
import Response from './Response'

axios.get('/my-route').then((response)=>{
    response = new Response(response)
    response.isInvalid
    response.isInformational
    response.isSuccessful
    ...
})
```

## Response Class
```
export default class Response {

    constructor(response) {
        this.data = response.data
        this.headers = response.headers
        this.status = response.status
    }

    /**
     * Is response invalid?
     * @see https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html
     * @return {Boolean}
     */
    get isInvalid() {
        return this.status < 100 || this.status >= 600
    }

    /**
     * Is response informative?
     * @return {Boolean}
     */
    get isInformational() {
        return this.status >= 100 && this.status < 200
    }

    /**
     * Is response successful?
     * @return {Boolean}
     */
    get isSuccessful() {
        return this.status >= 200 && this.status < 300
    }

    /**
     * Is the response a redirect?
     * @return {Boolean}
     */
    get isRedirection() {
        return this.status >= 300 && this.status < 400
    }

    /**
     * Is there a client error?
     * @return {Boolean}
     */
    get isClientError() {
        return this.status >= 400 && this.status < 500
    }

    /**
     * Was there a server side error?
     * @return {Boolean}
     */
    get isServerError() {
        return this.status >= 500 && this.status < 600
    }

    /**
     * Is the response OK?
     * @return {Boolean}
     */
    get isOk() {
        return 200 === this.status
    }

    /**
     * Is the client unauthorized?
     * @return {Boolean}
     */
    get isUnAuthorized() {
        return this.status === 401
    }

    /**
     * Is the response forbidden?
     * @return {Boolean}
     */
    get isForbidden() {
        return 403 === this.status
    }

    /**
     * Is the response a not found error?
     * @return {Boolean}
     */
    get isNotFound() {
        return 404 === this.status
    }

    /**
     * Is the response a redirect of some form?
     * @return {Boolean}
     */
    get isRedirect() {
        return [201, 301, 302, 303, 307, 308].includes(this.status)
    }
}
```