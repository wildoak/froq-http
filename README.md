# froq-http

![npm](https://img.shields.io/npm/v/froq-http.svg)
![node](https://img.shields.io/node/v/froq-http.svg)

![Travis branch](https://img.shields.io/travis/wildoak/froq-http/master.svg)
![Coveralls github](https://img.shields.io/coveralls/github/wildoak/froq-http.svg)

![license](https://img.shields.io/github/license/wildoak/froq-http.svg)
![GitHub tag](https://img.shields.io/github/tag/wildoak/froq-http.svg)
![GitHub issues](https://img.shields.io/github/issues/wildoak/froq-http.svg)
![GitHub last commit](https://img.shields.io/github/last-commit/wildoak/froq-http.svg)
![GitHub top language](https://img.shields.io/github/languages/top/wildoak/froq-http.svg)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/wildoak/froq-http.svg)

<img src="froq.png" width="100" alt="froQ logo" />

- [npm](https://www.npmjs.com/package/froq-http)
- [GitHub](https://github.com/wildoak/froq-http)

`froq-http` is not a framework for production, but for testing, creating mocks for integration tests on the fly. At current, https implemention is missing.

## Usage

`npm install froq-http`

We use npm package `debug`. To make me verbose use `DEBUG=froq-http`.


### Create an HTTP Server

```js
import http from 'froq-http';
const server = await http();
```

### Simple Route

```js
    
server
    .rest `/news`
    .type('json')
    .respond({test: true})
;

const result = await fetch(server.url('/news'));
const json = await result.json();
// {test: true}

await server.stop();
```


### Route with templates

```js
server
    .rest `/news/${'id'}`
    .respond(({result}) => {

        // result[0]: '12345'
        // result.id: '12345'

        return http.resp({
            type: 'json',
            body: result[0]
        });
    })
;

const result = await fetch(server.url('/news/12345'));
const json = await result.json();
// '12345'

await server.stop();
```


### Proxy Server

```js
// parallel
const [server1, server2] = await Promise.all([
    http('server1'),
    http('server2')
]);

server2
    .rest `/api/docs`
    .type('json')
    .respond([
        'doc1',
        'doc2'
    ]);

server1
    .rest `/index.html`
    .type('html')
    .respond('<html>...</html>')

    .rest `/${'*'}`
    .proxy(server2)
;


{
    // direct server1
    const result = await fetch(server1.url('/index.html'));
    const text = await result.text();
    // '<html>...</html>'
}

{
    // indirect server2 via server1
    const result = await fetch(server1.url('/api/docs'));
    const json = await result.json();
    // ['doc1', 'doc2']
}

await Promise.all([server1.stop(), server2.stop()]);
```


### Route with Error

```js
server
    .rest `/api/error`
    .respond(() => http.resp({
        status: 404,
        type: 'json',
        body: 'Not Found'
    }))
;

const result = await fetch(server.url('/api/error'));
// result.status: 404
// await result.json(): 'Not Found'

await server.stop();
```


### Simple Server implementation

```js
server
    .rest `/api/endpoint`
    .type('json')
    .respond(({body}) => ({
        a: body.a + 1,
        b: body.b + 'b'
    }))
;

const result = await fetch(server.url('/api/endpoint'), {
    method: 'POST',
    headers: {
        'content-type': 'application/json'
    },
    body: JSON.stringify({
        a: 1,
        b: 'b'
    })
});

const json = await result.json();
// {a: 2, b: 'bb'}

await server.stop();
```


### Response types

```js
server
    .type('json')

    .rest `/api/json`
    .respond({
        json: true
    })

    .rest `/api/html`
    .type('text/html')
    .respond('<p>Html</p>')

    .rest `/api/txt`
    .respond(http.resp({type: 'txt', body: 'This is a text.'}))
;

{
    // json
    const result = await fetch(server.url('/api/json'));
    // result.headers['content-type']: 'application/json'

    const json = await result.json();
    // {json: true}
}

{
    // html
    const result = await fetch(server.url('/api/html'));
    // result.headers['content-type']: 'text/html'
    
    const text = await result.text();
    // '<p>Html</p>'
    
    
}

{
    // text
    const result = await fetch(server.url('/api/txt'));
    // result.headers['content-type']: 'text/plain'

    const text = await result.text();
    // 'This is a text.'
}

await server.stop();
```


### Micro Login Server

```js
server
    .rest `/login`
    .type('json')
    .respond(({body}) => {
        if (body.username === 'user' && body.password === 'pass') {
            return {
                login: true
            };
        }

        return http.resp({
            status: 401,
            body: {
                login: false
            }
        });
    })
;

{
    // successful login
    const response = await fetch(server.url('/login'), {
        method: 'POST',
        headers: {
            'content-type': 'application/json'
        },
        body: JSON.stringify({
            username: 'user',
            password: 'pass'
        })
    });

    const json = await response.json();
    // {login: true}
}

{
    // login failed
    const response = await fetch(server.url('/login'), {
        method: 'POST',
        headers: {
            'content-type': 'application/json'
        },
        body: JSON.stringify({
            username: 'invalid_user',
            password: 'invalid_pass'
        })
    });

    const json = await response.json();
    // {login: false}
}


await server.stop();
```