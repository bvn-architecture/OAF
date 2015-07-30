## What OAF does

OAF makes it easy and fast to get to the image you want.

### What Open Asset does

[Open Asset](https://www.google.com/url?q=https%3A%2F%2Fopenasset.com%2F&sa=D&sntz=1&usg=AFQjCNHqyVtGTYw2y-dN1XHU_sX8KUEzWQ) is a proprietary system that stores photos and other assets for AEC companies.

It has a [REST API](http://www.google.com/url?q=http%3A%2F%2Fhelp.openasset.com%2F06_Integration%2FREST_API&sa=D&sntz=1&usg=AFQjCNFqWh20XqgvH9emcgP1pogzuucs6g) that allows developers to access and manipulate the database but that API is hard to use and slow.

## Why OAF exists

The OA REST API (look at all those acronyms!) requires [4 round trips to the server to retrieve an image](http://www.google.com/url?q=http%3A%2F%2Fstackoverflow.com%2Fquestions%2F22676924%2Fhow-to-construct-image-url-from-axomic-openasset-rest-api&sa=D&sntz=1&usg=AFQjCNEPUkki68k2sD7HFsu6alaLuNC1Uw). This was unacceptably slow for our needs. It also needed code that was overly complex for a regular person* to write. (*i.e. a non-developer.)

The OA REST API has a very direct mapping to the guts of the database. OAF exists to abstract away the details of that database and phrase the requests in terms of what people want, not how to get it.

As well as abstracting the API provided by Open Asset to make it easier to call, it also has a caching layer. The caching reduces the time take to fetch an image from about 5 seconds to regular image GET timing. I.e. about 125ms to get an image that OAF has reached vs 5000ms to get an image that OAF hasn’t cached. 

We think that a normalised API is a bad thing. [Others agree](http://www.google.com/url?q=http%3A%2F%2Fwww.soatothecloud.com%2F2012%2F11%2Fapi-server-design-making-de.html&sa=D&sntz=1&usg=AFQjCNHQS65cGqxJzDpp6yrcQ13KkZ18Ng) -- tldr: making multiple requests for a single image sucks.

OAF is an acronym for Open Asset Fixer.

## How to use OAF

### Some use cases

- As a marketing person I want to be able to put the best photo of a project onto a web page so that I can get a submission done quickly and not be exposed to risk.
- As a web developer I want to be able to write a page that has a grid of the best photos for all the projects.
- As a web developer I want to be able to show a project photo on a web page along with some of its meta data so that we can fulfil copyright obligations to the photographer.
- As an architect I want to be able to put several photos into a blog post so that I can illustrate a story about the project I worked on.

### Examples

To get an image you can either use a 303 redirect or a JSON request.

#### 303

To construct the request:

`/:project_number/:tag/:index/:size`

So in the real world that would be:

`http://oaf.com/S0910004/main/0/web_view`

That would get you the highest rated image from the S0910004 project, with the webview size.

The options for each part are:

##### Project_number

Not much in the way of option here! You either have that project or you don’t!

##### Tag

This one is kind of secret sauce. We rolled our own tagging system to allow freeform tagging. You can use if if you like, it’s pretty cool, but it’s not documented here yet (soon). Just set this to main and you’ll be fine.

##### Index

This tells OAF which photo to get. The photos need to be ranked using the OA system. The best image is at index 0, second best at 1 etc. If there is a clash (two photos ranked best) then it’s pot luck which one you’ll get.

##### Size

Size must be one of: square, thumbnail, small, web_view or medium. The actual size that you’ll get back is defined in your OA settings somewhere.

#### JSON

One day my prince will come...

## What’s inside OAF

### Architectural overview

The central concept used in OAF is that of Data Access Objects ([DAO]([https://en.wikipedia.org/wiki/Data_access_object](https://www.google.com/url?q=https%3A%2F%2Fen.wikipedia.org%2Fwiki%2FData_access_object&sa=D&sntz=1&usg=AFQjCNGF-JuHSomsJVDJiYpNGpmKTDTuDQ))):

In short, a DAO is an object that abstracts over the ‘accessing data’ part and presents a simple object oriented way to work with the objects. The objects in question are defined by the Open Asset API’s exposed resources [listed here]([http://help.openasset.com/06_Integration/REST_API#Resources](http://www.google.com/url?q=http%3A%2F%2Fhelp.openasset.com%2F06_Integration%2FREST_API%23Resources&sa=D&sntz=1&usg=AFQjCNG-hg0FFYjvbE-D1aiESafRCcvCCQ)).

When accessing an image for a project we need to send a get request to:

- `/Projects` - to get the images associated with a project number (code in OA parlance)
- `/Files` - to get the image sizes associated with an image
- `/Files/:id/Sizes` - to get the url a particular size is located at

When encapsulated using DAO objects accessing a particular project’s image url looks like:

```
        p = get_project()

        p.images[0][‘web_view’].url
```

In order to make this happen the models need a way to access the data remotely. This is done with two entwined super classes, Model and DataSource, intended to be inherited from by implementations. Inside of the Project model, for example, the images method is implemented like so:

```
        class Model
                …
                def get query
                        @data_source.get query =&gt; @oa_id
                   end
                ...
        end

        class Project < Model
                …
                def images
                        get :image
                end
                ...
        end
```

Note that the data source is passed in to the model’s constructor.        

- core
  - datasource
  - model
- models
  - Project
  - Image
- Size
  - data sources
- Open Asset - uses rest_client gem to fetch from OA’s API
  - Redis - uses redis gem to cache results
  - Double - combines two above data sources into one which will automatically fetch from redis if it exists, otherwise fetch from OA then cache the result in redis

- controller

- get redirect `/:project_number/:tag/:index/:size`
- get json `/:project_number.json`

#### Clever (in a bad way) Code Shortcut

The data source accepts a dictionary of symbol to id (e.g. `{ :project =&gt; ‘S0910004’ }`). The symbol is turned into a string, capitalized, then passed to Kernel.const_get to turn it into a reference to the class Project. If a model ever had a name that couldn’t be easily transformed in the above way to it’s constant representation then the data source would not work on it. It also requires that the models not be in a separate module namespace.

This is done to simplify things. However a more robust and less ‘clever’ way of doing things would be to have a dictionary somewhere of symbols to classes, e.g.:

```
class_map = {

  project: Project,

  image: Image,

  other_class: Namespace::OtherClass

}
```

## Coming soon Extra things to do

##### JSON API

- make it less impotent (i.e. include images in the to_json). NB: Projects#to_json was neutered because to_json is used to store it in redis. Using the to_json options parameter to tell it whether or not to include images in it’s json representation would be a good way to go.

##### Models

- add photographer, copyright holder, etc
