# Making HTTP REST Request in C++

## Introduction

Today, I am going to show you how to make HTTP request to a REST server using [C++ Requests](https://github.com/whoshuu/cpr) library by Huu Nguyen. Mr Nguyen is heavily influenced by [Python Requests](http://docs.python-requests.org/en/master/) design philosophy when writing [C++ Requests](https://github.com/whoshuu/cpr). Those who had used or are familiar with Python Requests, should feel right at home with [C++ Requests](https://github.com/whoshuu/cpr).

To demonstrate our client code, we need a web server that we can make our request to so in our case, we&#39;ll use ASP.NET Web API version 2 to implement our CRUD API. The web server is not this article&#39;s focus but I shall still devote some time to explain the Web API code. For those readers not interested in the server code (because they are not using ASP.NET), they can skip to the client section.

## ASP.NET Web API

I am not going to go through the details on how to setup the ASP.NET Web API project. Interested readers can read this [tutorial](https://docs.microsoft.com/en-us/aspnet/web-api/overview/getting-started-with-aspnet-web-api/tutorial-your-first-web-api) and that [tutorial](https://docs.microsoft.com/en-gb/aspnet/core/tutorials/first-web-api?view=aspnetcore-2.0) provided by Microsoft.

The Web API is based loosely on MVC design. MVC stands for Model, View and Controller. Model represents the data layer, usually they are classes that model after data design in storage, View represents the presentation layer while Controller is the business logic layer. Strictly speaking, a pure ASP.NET Web API server does not serve out HTML pages, so it does not have the presentation layer. In our example, we have the `Product` class as our data Model.

```Cpp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public int Qty { get; set; }
    public decimal Price { get; set; }
}
```

In our `ProductsController`, `Product` is stored in a `static Dictionary` which is not persistent, meaning to say the data disappears after the Web API server shuts down. But this should suffice for our demonstration without involving the use of database.

```Cpp
public class ProductsController : ApiController
{
    static Dictionary<long, Product> products = new Dictionary<long, Product>();
```

This is the `Create` method to create a `Product` object to store in our `dictionary`. `[FromBody]` attribute means the `Product` item shall be populated with the contents found in the request body.

```Cpp
[HttpPost]
public IHttpActionResult Create([FromBody] Product item)
{
    if (item == null)
    {
        return BadRequest();
    }

    products[item.Id] = item;

    return Ok();
}
```

To test our code, I use `curl` command. If you have already had [Postman](https://www.getpostman.com/) installed, you can use that as well. I am old school, so I prefer to use `curl` command directly.

```
curl -XPOST http://localhost:51654/api/products/create 
-H &#39;Content-Type: application/json&#39; -d&#39;{"Id":1, "Name":"ElectricFan","Qty":14,"Price":20.90}&#39;
```

* `-X` specifies the [HTTP verb](http://www.restapitutorial.com/lessons/httpmethods.html), `POST` which corresponds to `create` or `update` method
* 2<sup>nd</sup> argument is the URL to which this request should go to
* `-H` specifies the HTTP headers. We send JSON `string` so we set &#39;`Content-Type`&#39; to &#39;`application/json`&#39;.
* `-d` specifies the content body of the request. It can be seen clearly that the keys in the `json dictionary` correspond exactly to the `Product` members.


The output returned by `curl` is empty when the `post` request is successful. To see our created `Product`, we need to have the retrieval method which is discussed shortly below.

The methods to retrieve all products and a single `product` are listed below.<br />
__Note__: HTTP `GET` verb is used for data retrieval.

```Cpp
[HttpGet]
public List<Product> GetAll()
{
    List<Product> temp = new List<Product>();
    foreach(var item in products)
    {
        temp.Add(item.Value);
    }
    return temp;
}

[HttpGet]
public IHttpActionResult GetProduct(long id)
{
    try
    {
        return Ok(products[id]);
    }
    catch (System.Collections.Generic.KeyNotFoundException)
    {
        return NotFound();
    }
}
```

The respective `curl` commands retrieve all and a single `Product` based on id (which is `1`). The commandline argument is similar to what I have explained above, so I skip them.

```
curl -XGET http://localhost:51654/api/products&#39;

curl -XGET http://localhost:51654/api/products/1&#39;
```

The output is:

```
[{"Id":1,"Name":"ElectricFan","Qty":14,"Price":20.90}]

{"Id":1,"Name":"ElectricFan","Qty":14,"Price":20.90}
```

We see the 1<sup>st</sup> output is enclosed by `[]` because 1<sup>st</sup> command returns a collection of `Product` objects but in our case, we only have 1 `Product` right now.

Lastly, we have the `Update` and `Delete` method. The difference between HTTP `POST` and `PUT` verbs is that `PUT` is purely an `update` method whereas `POST` creates the object if it does not exist but `POST` can used for updating as well.

```Cpp
[HttpPut]
public IHttpActionResult Update(long id, [FromBody] Product item)
{
    if (item == null || item.Id != id)
    {
        return BadRequest();
    }

    if(products.ContainsKey(id)==false)
    {
        return NotFound();
    }
    var product = products[id];

    product.Name = item.Name;
    product.Qty = item.Qty;
    product.Price = item.Price;

    return Ok();
}
[HttpDelete]
public IHttpActionResult Delete(long id)
{
    var product = products[id];
    if (product == null)
    {
        return NotFound();
    }

    products.Remove(id);

    return Ok();
}
```

The respective `curl` commands below updates and deletes `Product` correspond to `id=1`.

```
curl -XPUT http://localhost:51654/api/products/1 
-H &#39;Content-Type: application/json&#39; -d&#39;{"Id":1, "Name":"ElectricFan","Qty":15,"Price":29.80}&#39;

curl -XDELETE http://localhost:51654/api/products/1
```

To see the `Product` is really updated or deleted, we have to use the retrieval `curl` command shown above.

## C++ Client Code

At last, we have to come to the main focus of this article! To able to use C++ Requests, please clone or download it [here](https://github.com/whoshuu/cpr) and include its `cpr` header. Alternatively, for Visual C++ users, you can install C++ Requests via [vcpkg](https://github.com/Microsoft/vcpkg). C++ Requests is abbreviated as cpr in vcpkg.

```
.\vcpkg install cpr

```

```Cpp
#include <cpr/cpr.h>
```

To send a `POST` request to create a `Product`, we put our `Product` json in [raw string literal](http://en.cppreference.com/w/cpp/language/string_literal) inside the `cpr::Body`. Otherwise, without raw string literal, we have to escape all the double quotes found in our json `string`.

```Cpp
auto r = cpr::Post(cpr::Url{ "http://localhost:51654/api/products/create" },
    cpr::Body{ R"({"Id":1, "Name":"ElectricFan","Qty":14,"Price":20.90})" },
    cpr::Header{ { "Content-Type", "application/json" } });
```

Compare this C++ code to raw `curl` command, we can see which information goes to where.

```
curl -XPOST http://localhost:51654/api/products/create 
-H &#39;Content-Type: application/json&#39; -d&#39;{"Id":1, "Name":"ElectricFan","Qty":14,"Price":20.90}&#39;

```

After product creation, we try to retrieve it using the C++ code below:

```Cpp
auto r = cpr::Get(cpr::Url{ "http://localhost:51654/api/products/1" });
```

The output is the same as the one from `curl` command which isn&#39;t strange since C++ Requests utilize `libcurl` underneath to do its work.

```json
{"Id":1,"Name":"ElectricFan","Qty":14,"Price":20.90}
```

The full C++ code to do CRUD with ASP.NET Web API is listed below with its output. By the way, CRUD is short for Create, Retrieve, Update and Delete. Be sure your ASP.NET Web API is up and running before running the C++ code below.

```Cpp
int main()
{
    {
        std::cout << "Action: Create Product with Id = 1" << std::endl;
        auto r = cpr::Post(cpr::Url{ "http://localhost:51654/api/products/create" },
            cpr::Body{ R"({"Id":1, 
            "Name":"ElectricFan","Qty":14,"Price":20.90})" },
            cpr::Header{ { "Content-Type", "application/json" } });
        std::cout << "Returned Status:" << r.status_code << std::endl;
    }
    {
        std::cout << "Action: Retrieve the product with id = 1" << std::endl;
        auto r = cpr::Get(cpr::Url{ "http://localhost:51654/api/products/1" });
        std::cout << "Returned Text:" << r.text << std::endl;
    }
    {
        std::cout << "Action: Update Product with Id = 1" << std::endl;
        auto r = cpr::Post(cpr::Url{ "http://localhost:51654/api/products/1" },
            cpr::Body{ R"({"Id":1, 
            "Name":"ElectricFan","Qty":15,"Price":29.80})" },
            cpr::Header{ { "Content-Type", "application/json" } });
        std::cout << "Returned Status:" << r.status_code << std::endl;
    }
    {
        std::cout << "Action: Retrieve all products" << std::endl;
        auto r = cpr::Get(cpr::Url{ "http://localhost:51654/api/products" });
        std::cout << "Returned Text:" << r.text << std::endl;
    }
    {
        std::cout << "Action: Delete the product with id = 1" << std::endl;
        auto r = cpr::Delete(cpr::Url{ "http://localhost:51654/api/products/1" });
        std::cout << "Returned Status:" << r.status_code << std::endl;
    }
    {
        std::cout << "Action: Retrieve all products" << std::endl;
        auto r = cpr::Get(cpr::Url{ "http://localhost:51654/api/products" });
        std::cout << "Returned Text:" << r.text << std::endl;
    }

    return 0;
}
```

The output as mentioned is shown below. I only display the returned text when the CRUD supports it, otherwise I just display the status. HTTP Status 200 means successful HTTP request. For example, Create/Update/Delete operation does not return any text, so I just display their status.

```
Action: Create Product with Id = 1
Returned Status:200

Action: Retrieve the product with id = 1
Returned Text:{"Id":1,"Name":"ElectricFan","Qty":14,"Price":20.90}

Action: Update Product with Id = 1
Returned Status:200

Action: Retrieve all products
Returned Text:[{"Id":1,"Name":"ElectricFan","Qty":15,"Price":29.80}]

Action: Delete the product with id = 1
Returned Status:200

Action: Retrieve all products
Returned Text:[]
```

For users looking to send request with parameters like below, you can make use of the `cpr::Parameters`.

```
http://www.example.com/products?quota=500&sold=true
```

C++ code for the above URL example is as follows:

```Cpp
auto r = cpr::Get(cpr::Url{ "http://www.example.com/products" },
    cpr::Parameters{{"quota", "500"}, {"sold", "true"}});
```

## History

* 28<sup>th</sup> June, 2022: Fixed the Newtonsoft.Json vulnerability reported by Github in RestWebApp project.
* 17<sup>th</sup> May, 2018: First release.
