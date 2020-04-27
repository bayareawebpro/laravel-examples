## Laravel Nova Resource Request Helper Vue Mixin

A Vue Mixin for using the Nova Resource Routes for other purposes by mapping Resource Responses to normal objects / arrays.

```javascript
this.fetchResourceEntity('media',id).then((entity)=>{
    this.entity = entity
})
```

```javascript
this.fetchResourceCollection('media', {
    orderByDirection: this.sort,
    search: this.searchTerm,
    orderBy: this.orderBy,
    page: this.page,
    perPage: 50,
})
.then((entities) => {
    this.entities = entities
})
```


```javascript
this.uploadResource('media', {file}).then((entity) => {
    this.$toasted.show(this.__('Uploaded: :name',entity), {
        type: 'success'
    })
})
```

```javascript
export default {
    methods:{
        /**
         * Map Nova Resource to Normal Object Format Including Additional Properties.
         * @param resource {Object}
         * @return {Object}
         */
        resourceToObject({fields}) {
            return fields.reduce((obj, item) => {
                obj[item.attribute] = item.value
                if(item.hasOwnProperty('thumbnailUrl')){
                    obj.url = item.thumbnailUrl
                }else if(item.hasOwnProperty('previewUrl')){
                    obj.url = item.previewUrl
                }
                return obj
            }, {})
        },
        /**
         * Fetch Resource Entity
         * @param resourceKey {String}
         * @param id {String|Number}
         * @return {Promise<Array>}
         */
        async fetchResourceEntity(resourceKey, id){
            return await Nova.request()
                .get(`/nova-api/${resourceKey}/${id}`)
                .then(({data})=>this.resourceToObject(data.resource))
                .catch(this.handleResourceError)
        },
        /**
         * Fetch Resource Collection
         * @param resourceKey {String}
         * @param params {Object}
         * @return {Promise<Array>}
         */
        async fetchResourceCollection(resourceKey, params){
            return await Nova.request()
                .get(`/nova-api/${resourceKey}`,{params})
                .then(({data})=>data.resources.map((resource)=>this.resourceToObject(resource)))
                .catch(this.handleResourceError)
        },
        /**
         * Fetch Resource Entity as Object
         * @param resourceKey {String}
         * @param params {Object}
         * @return {Promise<Array>}
         */
        async uploadResource(resourceKey, params){
            const data = new FormData
            Object.entries(params).forEach(([key,value])=>data.append(key, value))
            return await Nova.request()
                .post(`/nova-api/${resourceKey}`,data)
                .then(({data})=>data)
                .catch(this.handleResourceError)
        },
        /**
         * Handle Resource Error
         * @param error {Error}
         * @return void
         */
        handleResourceError(error){
            this.$toasted.show(this.__(':message',{message:error.toString()}),{
                type: 'error'
            })
        }
    }
}
```