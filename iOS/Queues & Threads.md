#### Dispatch Queues

Creating a dispatch queue is pretty simple on your part, as you can see in the example below:

```swift
let label = "com.yourcompany.mycoolapp.networking"
let queue = DispatchQueue(label: label)
```

When your app starts, a `main` dispatch queue is automatically created for you. It’s a serial queue that’s responsible for your UI. Because it’s used so often, Apple has made it available as a class variable, which you access via `DispatchQueue.main`. **You *never* want to execute something synchronously against the main queue, unless it’s related to actual UI work**. Otherwise, you’ll lock up your UI which could potentially(可能) degrade your app performance.

In order to create a concurrent queue, simply pass in the `.concurrent` attribute, like so:

```swift
let label = "com.yourcompany.mycoolapp.networking"
let queue = DispatchQueue(label: label, attributes: .concurrent)
```

#### Quality of Service

When using a concurrent dispatch queue, you’ll need to tell iOS how important the tasks are that get sent to the queue so that it can properly prioritize(优先) the work that needs to be done

```swift
let queue = DispatchQueue.global(qos: .userInteractive)
```

The `.userInteractive` QoS is recommended for tasks that the user *directly interacts* with. UI-updating calculations, animations or anything needed to keep the UI responsive and fast.

### Adding Task to Queues

doesn’t need to happen immediately and depends on networking I/O, so you should send it to the global utility queue:

```swift
DispatchQueue.global(qos: .utility).async {
  guard let self else { return }

  // Perform your work here
  // ...

  // Switch back to the main queue to
  // update your UI
  DispatchQueue.main.async {
    self.label = "New articles available!"
  }
}
```

For instance, if you make a network request from a view controller that has been dismissed in the meantime, the closure will still get called. If you capture the view controller weakly, it will be `nil`. However, if you capture it strongly, the view controller will remain alive until the closure finishes its work. Keep that in mind and capture weakly or strongly based on your needs.

## Image Loading Example

When the view loads, it just grabs a static list of image URLs to be displayed. In a production app, of course, you’d likely be making a network call at this point to generate a list of items to display, but for this example it’s easier to hardcode a list of images.

The proper choice here is to use the `.utility` queue. You’ve got no control over how long a network call will take to complete and you want the OS to properly balance the speed vs. battery life of the device.

```swift
private func downloadWithUrlSession() {
  URLSession.shared.dataTask(with: url) { data, _, _ in
    guard let data, let uiImage = UIImage(data: data) else {
      return
    }

    DispatchQueue.main.async {
      image = Image(uiImage: uiImage)
    }
  }
  .resume()
}

```

## Using `DispatchWorkItem`

There’s another way to submit work to a `DispatchQueue` besides passing an anonymous closure. **`DispatchWorkItem` is a class that provides an actual object to hold the code you wish to submit to a queue.**

```swift
let queue = DispatchQueue(label: "xyz")
let workItem = DispatchWorkItem {
  print("The block of code ran!")
}
queue.async(execute: workItem)
```

The `DispatchWorkItem` class also provides a `notify(queue:execute:)` method which can be used to identify(识别) another `DispatchWorkItem` that should be executed after the current work item completes.

```swift
let queue = DispatchQueue(label: "xyz")
let backgroundWorkItem = DispatchWorkItem { }
let updateUIWorkItem = DispatchWorkItem { }

backgroundWorkItem.notify(queue: DispatchQueue.main,
                          execute: updateUIWorkItem)
queue.async(execute: backgroundWorkItem)
```









