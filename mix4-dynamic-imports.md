
# Laravel Mix4 Dynamic Imports
You cannot use vendor extract with this method, it's one or the other.
You can extract your own vendor files with this method in a more robust manner.

### NPM Dev Package
```json
{
"@babel/plugin-syntax-dynamic-import": "^7.2"
}
```

### Mix Plugin
```javascript
mix.babelConfig({
    plugins: ['@babel/plugin-syntax-dynamic-import'],
})
```

### Vendor Import
```javascript
const initEcho = async () =>{
    const imported = await import(/*webpackChunkName:"broadcasting"*/"./EchoService")
    imported.default
}
```

### Component Import
```javascript
const routes = [
    {
        name: 'account.edit', path: '/account/edit', meta: {middleware: ['auth']},
        component: () => import(/*webpackChunkName:"account"*/"@page/Account/Edit"),
    }
]
```