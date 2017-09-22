+++
title = "Understand Json Patch"
date = 2017-09-22T16:42:11+08:00
tags = ["blog", "uptoknow", "json patch"]
categories = ["json"]
+++

When we update an resource using the API, we generially will first get the resource, and then update it, and put back the entire object, this can waste bandwidth and processtime for large resources. Another choice is we can use [HTTP PATCH](https://tools.ietf.org/html/rfc5789) to send an json patch to the api server.

There are two kinds of patch you can use, [JSON PATCH](https://tools.ietf.org/html/rfc6902) and [JSON Merge Patch](https://tools.ietf.org/html/rfc7386). Both have cons and pros.

# JSON Patch

The JSOn Patch format is similar to a database transaction: it is an array of mutating operations on a JSON object, and will be executed automically by a proper implementation. It is a series of "add", "remove", "copy" operations.

Here is an JSON exmaple:

```
{
	"users" : [
		{ "name" : "haoran" , "email" : "smarterhaoran@gmail.com" },
		{ "name" : "foo" , "email" : "foo@example.org" }
	]
}
```

If we want to change the email of user "haoran" to "uptoknow@gmail.com", we can sent a patch to the api server with body:

```
[
	{
		"op" : "replace" ,
		"path" : "/users/0/email" ,
		"value" : "uptoknow@gmail.com"
	}
]
```
we need remember using the header json-patch+json when you sent the request:

```
curl -X PATCH -H "Content-Type: application/json-patch+json" \
-D '[{"op":"replace","path":"/users/0/email","value":"uptoknow@gmail.com"}]' http://<server_ip>/api
```

# JSON Merge Patch

Alongside JSON Patch there is an other JSON-based format, [JSON Merge Patch - RFC 7386](https://tools.ietf.org/html/rfc7386) , which can be used more or less for the same purpose, ie. it describes a changed version of a JSON document. The conceptual difference compared to JSON Patch is that JSON Merge Patch is similar to a diff file. It simply contains the nodes of the document which should be different after execution.

As a quick example (taken from the spec) if we have the following document:

```
{
	"a": "b",
	"c": {
		"d": "e",
		"f": "g"
	}
}
```

Then we can run the following patch on it:

{
	"a":"z",
	"c": {
		"f": null
	}
}

which will change the value of "a" to "z" and will delete the "f" key.

The simplicity of the format may look first promising at the first glance, since most probably anyone understanding the schema of the original document will also instantly understand a merge patch document too. It is just a standardization of one may naturally call a patch of a JSON document.

But this simplicity comes with some limitations:

* Deletion happens by setting a key to null. This inherently means that it isn?t possible to change a key?s value to null, since such modification cannot be described by a merge patch document.
* Arrays cannot be manipulated by merge patches. If you want to add an element to an array, or mutate any of its elements then you have to include the entire array in the merge patch document, even if the actually changed parts is minimal.
* the execution of a merge patch document never results in error. Any malformed patch will be merged, so it is a very liberal format. It is not necessarily good, since you will probably need to perform programmatic check after merge, or run a JSON Schema validation after the merge.

# Summary

JSON Merge Patch is a naively simple format, with limited usability. Probably it is a good choice if you are building something small, with very simple JSON Schema, but you want offer a quickly understandable, more or less working method for clients to update JSON documents. A REST API designed for public consumption but without appropriate client libraries might be a good example.

For more complex usecases I?d pick JSON Patch, since it is applicable to any JSON documents (unline merge patch, which is not able to set keys to null). The specification also ensures atomic execution and robust error reporting.


Original ref link: http://erosb.github.io/post/json-patch-vs-merge-patch/
