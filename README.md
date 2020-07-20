# ytel
`ytel` is an experimental YouTube "frontend" for Emacs. It's goal is to allow the user to collect the results of a YouTube search in an elfeed-like buffer and then manipulate them with Emacs Lisp. The gif below shows that `ytel` can be used to play videos in an external player, to learn how to emulate it refer to the [usage](#usage) section below.

<p align="center">
  <img src="https://github.com/gRastello/ytel/blob/master/pic/demonstration.gif">
</p>

This project was inspired by [elfeed](https://github.com/skeeto/elfeed/) and [Invidious](https://github.com/omarroth/invidious) (it does indeed use the Invidious APIs).

## Installation
This project is on [MELPA](https://melpa.org/): you should be able to `M-x package-install RET ytel`. Another option is to clone this repository under your `load-path`.

### Dependencies
While `ytel` does not depend on any Emacs package it does depend on `curl` so, if you happen not to have it, install it through your package manager (meme distros aside it is probably in your repos).

## Usage
Once everything is loaded `M-x ytel` creates a new buffer and puts it in `ytel-mode`. Some of the ways you can interact with the buffer are shown below.

| key            | binding                     | description                                           |
|----------------|-----------------------------|-------------------------------------------------------|
| <kbd>n</kbd>   | `next-line`                 | Move cursor to next line                              |
| <kbd>p</kbd>   | `previous-line`             | Move cursor to previous line                          |
| <kbd>q</kbd>   | `ytel-quit`                 | Bury the `*ytel*` buffer                              |
| <kbd>s</kbd>   | `ytel-search`               | Make a new search                                     |
| <kbd>></kbd>   | `ytel-search-next-page`     | Go to next page                                       |
| <kbd><</kbd>   | `ytel-search-previous-page` | Go to previous page                                   |
| <kbd>t</kbd>   | `ytel-search-type`          | Change the type of results (videos, playlists, etc.). |
| <kbd>S</kbd>   | `ytel-sort-videos`          | Sort videos on the current buffer.                    |
| <kbd>Y</kbd>   | `ytel-yank-channel-feed`    | Copy the channel RSS feed for the current entry       |
| <kbd>RET</kbd> | `ytel-open-entry`           | Open entry                                            |

Pressing <kbd>RET</kbd> will act differently depending of the type of the current entry. If the entry is a playlist or a channel, it will open a new buffer with its videos.
Pressing `s` will prompt for some search terms and populate the buffer once the results are available. One can access information about a video via the function `ytel-get-current-video` that returns the video at point. Videos returned by `ytel-get-current-video` are cl-structures so you can access their fields with the `ytel-video-*` functions. Currently videos have seven fields:

| field       | description                                |
|-------------|--------------------------------------------|
| `id`        | the video's id                             |
| `title`     | the video's title                          |
| `author`    | name of the author of the video            |
| `authorId`  | id of the channel that updated the video   |
| `length`    | length of the video in seconds             |
| `views`     | number of views                            |
| `published` | date of publication (unix-style timestamp) |

With this information we can implement a function to stream a video in `mpv` (provided you have `youtube-dl` installed) as follows:
```elisp
(defun ytel-watch ()
  "Stream video at point in mpv."
  (interactive)
  (if (equal (ytel--get-entry-type (ytel-get-current-video)) 'video)
      (let* ((video (ytel-get-current-video))
	     (id    (ytel-video-id video)))
	(start-process "ytel mpv" nil
		       "mpv"
		       (concat "https://www.youtube.com/watch?v=" id
			       "--ytdl-format=bestvideo[height<=?720]+bestaudio/best"))
	(message "Starting streaming..."))
    (message "Not a video.")))
```

And bind it to a key in `ytel-mode` with
```elisp
(define-key ytel-mode-map "y" #'ytel-watch)
```
This is of course just an example. You can similarly implement functions to:
- open a video in the browser,
- download a video,
- download just the audio of a video,
- download a playlist
by relying on the correct external tool.

## Customization options
You can define default actions for `ytel-open-entry` depending on the type of entry by modifying the default-action variables, which are `ytel--default-video-action`, `ytel--default-playlist-action`, and `ytel--default-channel-action`. 
As an example, we can take `ytel-watch` and set it as a default action for videos, by binding it to `ytel--default-video-action`.
```elisp
(setf ytel--default-video-action #'ytel-watch)
```

It is also possible to customize the sorting criterion of the results by setting the variable `ytel-sort-criterion` to one of the following symbols `relevance`, `rating`, `upload_date` or `view_count`.
The default value is `relevance`.

Also, you can toggle the display of Unicode icons for some items in the `*ytel*` buffer by setting `ytel-show-fancy-icons` to `t`. Customization of these icons is done with `ytel-icons`.

## Potential problems and potential fixes
`ytel` does not use the official YouTube APIs but relies on the [Invidious](https://github.com/omarroth/invidious) APIs (that in turn circumvent YouTube ones). The variable `ytel-invidious-api-url` points to the invidious instance (by default `https://invidio.us`) to use; you might not need to ever touch this, but if [invidio.us](https://invidio.us) goes down keep in mind that you can choose [another instance](https://github.com/omarroth/invidious#invidious-instances). Moreover the default Invidious instance is generally speaking stable, but sometimes your `ytel-search` might hang; in that case `C-g` and retry.

Currently some wide unicode characters (such as Chinese/Japanese/Korean characters) are *very likely* to mess up the `*ytel*` buffer. The messing up is not that bad but things will not be perfectly aligned. Fixing this problem will most likely require a rewrite of how the ytel buffer is actually drawn. We're (somewhat) working on it.

## Contributing
Feel free to open an issue or send a pull request. I'm quite new to writing Emacs packages so any help is appreciated.

## FAQ

#### Why?
One can easily subscribe to YouTube channels via an RSS feed and access it in Emacs via [elfeed](https://github.com/skeeto/elfeed/) but sometimes I want to search YouTube for videos of people I don't necessarily follow (e.g. for searching a tutorial, or music, or wasting some good time) and being able to do that without switching to a browser is nice.

#### What about [helm-youtube](https://github.com/maximus12793/helm-youtube) and [ivy-youtube](https://github.com/squiter/ivy-youtube)?
First of all those packages require you to get a Google API key, while `ytel` uses the [Invidious](https://github.com/omarroth/invidious) APIs that in turn do not use the official Google APIs.

Moreover those packages are designed to select a YouTube search result and play it directly in your browser while `ytel` is really a way to collect search results in an `elfeed`-like buffer and make them accessible to the user via functions such as `ytel-get-current-video` so the user gets to decide what to to with them.
