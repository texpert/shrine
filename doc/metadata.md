# Extracting Metadata

Before a file is uploaded, Shrine automatically extracts metadata from it, and
stores them in the `Shrine::UploadedFile` object.

```rb
uploaded_file = uploader.upload(file)
uploaded_file.metadata #=>
# {
#   "size" => 345993,
#   "filename" => "matrix.mp4",
#   "mime_type" => "video/mp4",
# }
```

The following metadata is extracted by default:

| Key         | Default source                                     |
| :-----      | :------                                            |
| `filename`  | extracted from `io.original_filename` or `io.path` |
| `mime_type` | extracted from `io.content_type`                   |
| `size`      | extracted from `io.size`                           |

You can access extracted metadata in three ways:

```rb
# via methods (if they're defined)
uploaded_file.size
uploaded_file.original_filename
uploaded_file.mime_type

# via the metadata hash
uploaded_file.metadata["size"]
uploaded_file.metadata["filename"]
uploaded_file.metadata["mime_type"]

# via the #[] operator
uploaded_file["size"]
uploaded_file["filename"]
uploaded_file["mime_type"]
```

Under the hood, `Shrine#upload` calls `Shrine#extract_metadata`, which you can
also use directly to extract metadata from any IO object:

```rb
uploader.extract_metadata(io) #=>
# {
#   "size" => 345993,
#   "filename" => "matrix.mp4",
#   "mime_type" => "video/mp4",
# }
```

`Shrine#upload` accepts a `:metadata` option which accepts the following values:

  * `Hash` – adds/overrides extracted metadata with the given hash

    ```rb
    uploaded_file = uploader.upload(file, metadata: { "filename" => "Matrix[1999].mp4", "foo" => "bar" })
    uploaded_file.original_filename #=> "Matrix[1999].mp4"
    uploaded_file.metadata["foo"]   #=> "bar"
    ```

  * `false` – skips metadata extraction (useful in tests)

    ```rb
    uploaded_file = uploader.upload(file, metadata: false)
    uploaded_file.metadata #=> {}
    ```

  * `true` – forces metadata extraction when a `Shrine::UploadedFile` is being
    uploaded (by default metadata is simply copied over)

    ```rb
    uploaded_file = uploader.upload(uploaded_file, metadata: true)
    uploaded_file.metadata # re-extracted metadata
    ```

## MIME type

By default, the `mime_type` metadata will be copied over from the
`#content_type` attribute of the input file (if present). However, since
`#content_type` value comes from the `Content-Type` header of the upload
request, it's *not guaranteed* to hold the actual MIME type of the file (browser
determines this header based on file extension). Moreover, only
`ActionDispatch::Http::UploadedFile`, `Shrine::Plugins::RackFile::UploadedFile`,
and `Shrine::Plugins::DataUri::DataFile` objects have `#content_type` defined,
so, when uploading simple file objects, `mime_type` will be nil. That makes
relying on `#content_type` both a security risk and limiting.

To remedy that, Shrine comes with a
[`determine_mime_type`][determine_mime_type] plugin which is able to extract
the MIME type from IO *content*:

```rb
# Gemfile
gem "marcel", "~> 0.3"
```
```rb
Shrine.plugin :determine_mime_type, analyzer: :marcel
```
```rb
uploaded_file = uploader.upload StringIO.new("<?php ... ?>")
uploaded_file.mime_type #=> "application/x-php"
```

You can choose different analyzers, and even mix-and-match them. See the
[`determine_mime_type`][determine_mime_type] plugin docs for more details.

## Image Dimensions

Shrine comes with a `store_dimensions` plugin for extracting image dimensions.
It adds `width` and `height` metadata values, and also adds `#width`,
`#height`, and `#dimensions` methods to the `Shrine::UploadedFile` object. By
default, the plugin uses [FastImage] to analyze dimensions, but you can also
have it use [MiniMagick] or [ruby-vips]:

```rb
Shrine.plugin :store_dimensions, analyzer: :mini_magick
```
```rb
uploaded_file = uploader.upload(image)
uploaded_file.metadata["width"]  #=> 1600
uploaded_file.metadata["height"] #=> 900

# convenience methods
uploaded_file.width      #=> 1600
uploaded_file.height     #=> 900
uploaded_file.dimensions #=> [1600, 900]
```

## Custom metadata

In addition to the built-in metadata, Shrine allows you to extract and store
any custom metadata, using the `add_metadata` plugin (which extends
`Shrine#extract_metadata`). For example, you might want to extract EXIF data
from images:

```rb
require "mini_magick"

class ImageUploader < Shrine
  plugin :add_metadata

  add_metadata :exif do |io, context|
    Shrine.with_file(io) do |file|
      begin
        MiniMagick::Image.new(file.path).exif
      rescue MiniMagick::Error
        # not a valid image
      end
    end
  end
end
```
```rb
uploaded_file = uploader.upload(image)
uploaded_file.metadata["exif"] #=> {...}
uploaded_file.exif             #=> {...}
```

Or, if you're uploading videos, you might want to extract some video-specific
meatadata:

```rb
require "streamio-ffmpeg"

class VideoUploader < Shrine
  plugin :add_metadata

  add_metadata do |io, context|
    movie = Shrine.with_file(io) { |file| FFMPEG::Movie.new(file.path) }

    { "duration"   => movie.duration,
      "bitrate"    => movie.bitrate,
      "resolution" => movie.resolution,
      "frame_rate" => movie.frame_rate }
  end
end
```
```rb
uploaded_file = uploader.upload(video)
uploaded_file.metadata #=>
# {
#   ...
#   "duration" => 7.5,
#   "bitrate" => 481,
#   "resolution" => "640x480",
#   "frame_rate" => 16.72
# }
```

The yielded `io` object will not always be an object that responds to `#path`.
If you're using the `data_uri` plugin, the `io` will be a `StringIO` wrapper.
With `restore_cached_data` or `refresh_metadata` plugins, `io` might be a
`Shrine::UploadedFile` object. If you're using a metadata analyzer that
requires the source file to be on disk, you can use `Shrine.with_file` to
ensure you have a file object.

## Metadata columns

If you want to write any of the metadata values into a separate database column
on the record, you can use the `metadata_attributes` plugin.

```rb
Shrine.plugin :metadata_attributes, :mime_type => :type
```
```rb
photo = Photo.new(image: file)
photo.image_type #=> "image/jpeg"
```

## Direct uploads

When attaching files that were uploaded directly to the cloud or a [tus
server], Shrine won't automatically extract metadata from them, instead it will
copy any existing metadata that was set on the client side. The reason why this
is the default behaviour is because extracting the metadata would require (at
least partially) retrieving file content from the storage, which could
potentially be expensive depending on the storage and the type of metadata
being extracted.

There are two ways of extracting metadata from directly uploaded files. If you
want metadata to be automatically extracted on assignment (which is useful if
you want to validate the extracted metadata or have it immediately available
for any other reason), you can load the `restore_cached_data` plugin:

```rb
class ImageUploader < Shrine
  plugin :restore_cached_data # automatically extract metadata from cached files on assignment
end
```
```rb
photo.image = '{"id":"ks9elsd.jpg","storage":"cache","metadata":{}}' # metadata is extracted
photo.image.metadata #=>
# {
#   "size" => 4593484,
#   "filename" => "nature.jpg",
#   "mime_type" => "image/jpeg"
# }
```

On the other hand, if you're using backgrounding, you can extract metadata
during background promotion using the `refresh_metadata` plugin (which the
`restore_cached_data` plugin uses internally):

```rb
class ImageUploader < Shrine
  plugin :refresh_metadata
  plugin :processing

  # this will be called in the background if using backgrounding plugin
  process(:store) do |io, context|
    io.refresh_metadata!(context) # extracts metadata and updates `io.metadata`
    io
  end
end
```

If you have metadata that is cheap to extract in the foreground, but also have
additional metadata that can be extracted asynchronously, you can combine the
two approaches. For example, if you're attaching video files, you might want to
extract MIME type upfront and video-specific metadata in a background job, which
can be done as follows (provided that `backgrounding` plugin is used):

```rb
class MyUploader < Shrine
  plugin :determine_mime_type # this will be called in the foreground
  plugin :restore_cached_data
  plugin :refresh_metadata
  plugin :add_metadata
  plugin :processing

  # this will be called in the background if using backgrounding plugin
  process(:store) do |io, context|
    io.refresh_metadata!(context)
    io
  end

  add_metadata do |io, context|
    next unless context[:action] == :store # this will be the case during promotion

    Shrine.with_file(io) do |file|
      # example of metadata extraction
      movie = FFMPEG::Movie.new(file.path) # uses the streamio-ffmpeg gem

      { "duration"   => movie.duration,
        "bitrate"    => movie.bitrate,
        "resolution" => movie.resolution,
        "frame_rate" => movie.frame_rate }
    end
  end
end
```

If you want to do both metadata extraction and file processing during
promotion, you can wrap both in an `UploadedFile#open` block to make
sure the file content is retrieved from the storage only once.

```rb
class MyUploader < Shrine
  plugin :refresh_metadata
  plugin :processing

  process(:store) do |io, context|
    io.open do |io, context|
      io.refresh_metadata!(context)

      original = io.download # reuses already open uploaded file
      # ... processing ...
    end
  end
end
```

If you're dealing with large files, it's recommended to also use the `tempfile`
plugin to make sure the same copy of the uploaded file is used for metadata
extraction (`Shrine.with_file`) and processing (`UploadedFile#tempfile`).

```rb
Shrine.plugin :tempfile # load it globally so that it overrides `Shrine.with_file`
```
```rb
class MyUploader < Shrine
  plugin :refresh_metadata
  plugin :processing

  process(:store) do |io, context|
    io.open do |io, context|
      io.refresh_metadata!(context)

      original = io.tempfile # used the cached tempfile
      # ... processing ...
    end
  end
end
```

[`file`]: http://linux.die.net/man/1/file
[MimeMagic]: https://github.com/minad/mimemagic
[Marcel]: https://github.com/basecamp/marcel
[FastImage]: https://github.com/sdsykes/fastimage
[MiniMagick]: https://github.com/minimagick/minimagick
[ruby-vips]: https://github.com/libvips/ruby-vips
[tus server]: https://github.com/janko/tus-ruby-server
[determine_mime_type]: /doc/plugins/determine_mime_type.md#readme
