# AdvancedTouch
### Swift Advanced Touch Handling in iOS: Coalescing and Prediction

*Companion project to http://flexmonkey.blogspot.co.uk/2015/09/advanced-touch-handling-in-ios9.html*

![screenshot](AdvancedTouch/assets/screenshot.jpg)

With the introduction of the iPad Pro, low latency, high resolution touch handling has become more important than ever. Even if your app runs smoothly at 60 frames per second, high-end iOS devices have a touch scan rate of 120 hertz and with the standard out-of-the-box touch handling, you could easily miss half of your user’s touch movements.

Luckily, with iOS, Apple have introduced some new functionality to `UIEvent` to reduce touch latency and improve temporal resolution. There's a demo application to go with with post that you can find at my GitHub repository here.

### Touch Coalescing

The first feature is touch coalescing. If your app is sampling touches with an overridden `touchesMoved()`, the most amount of touch samples you’ll get is 60 per second and, if your app is concerning itself with some fancy calculations on the main thread, it could well be less than that.

Touch coalescing allows you to access all the intermediate touches that may have occurred between `touchesMoved()` invocations.  This allows, for example, your app to draw a smooth curve consisting of half a dozen points when you’ve only received one `UIEvent`.

The syntax is super simple. In my demo application, I have a few `CAShapeLayer` instances that I draw upon. The first layer (`mainDrawLayer`, which contains blue lines with circles at each vertex) is drawn on using information from the main `UIEvent` from `touchesMoved()`:

    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
        super.touchesMoved(touches, with: event)

        guard let touch = touches.first, let event = event else {
            return
        }

        let locationInView = touch.location(in: view)

        mainDrawPath.addLine(to: locationInView)
        mainDrawPath.append(UIBezierPath.createCircleAtPoint(origin: locationInView, radius: 4))
        mainDrawPath.move(to: locationInView)

        mainDrawLayer.path = mainDrawPath.cgPath
        [...]

If you’re wondering where `createCircleAtPoint()` method comes from, it’s a small extension I wrote to `UIBezierPath` that returns a circle path at a given point:

    extension UIBezierPath {
        static func createCircleAtPoint(origin: CGPoint, radius: CGFloat) -> UIBezierPath {
            let boundingRect = CGRect(x: origin.x - radius,
                y: origin.y - radius,
                width: radius * 2,
                height: radius * 2)

            let circle = UIBezierPath(ovalIn: boundingRect)

            return circle
        }
    }

If the user moves their finger across the screen fast enough, they’ll get a jagged line, which is probably not what they want. To access the intermediate touches, I use the `coalescedTouchesForTouch()` method of the event which returns an array of `UITouch`:

        [...]
        if let coalescedTouches = event.coalescedTouches(for: touch) {
            print("coalescedTouches:", coalescedTouches.count)

            for coalescedTouch in coalescedTouches {
                let locationInView = coalescedTouch.location(in: view)

                coalescedDrawPath.addLine(to: locationInView)
                coalescedDrawPath.append(UIBezierPath.createCircleAtPoint(origin: locationInView, radius: 2))
                coalescedDrawPath.move(to: locationInView)
            }

            coalescedDrawLayer.path = coalescedDrawPath.cgPath
        }
        [...]

Here, I loop over those touches (I trace the number of touches to the console for information) and create a yellow line by drawing on `coalescedDrawLayer` as an overlay.  I’ve added a horrible loop to the touchesMoved method to slow things down a bit:

        var foo = Double(1)

        for bar in 0 ... 4_000_000 {
            foo += sqrt(Double(bar))
        }

If you run this app on your iOS device and scribble away you can see the two results: a jagged blue line and a beautifully smooth yellow line based on all those intermediate touch events that could have been missed. At each touch location, I add a small circle which clearly illustrates the increased resolution of the yellow curve based on the coalesced data.

### Predictive Touch

Something maybe even cleverer is touch prediction which allows you to preempt where a user’s finger (or Apple Pencil) may be in the near future. Predictive touch uses some highly tuned algorithms and is continually updated with where iOS expects the users touch to be in approximately a frame in the future. This means you could begin preparing user interface components (e.g. beginning a fade or instantiating some objects) before they’re actually required to help reduce latency.

In my demo application, I display the predicted touch as small white spots with “tails” that originate for their predicting touch. The syntax is not dissimilar to that of coalesced touch: the event has a new method, `predictedTouchesForTouch()`, which returns an an array of `UITouch`:

        [...]
        if let predictedTouches = event.predictedTouches(for: touch) {
            print("predictedTouches:", predictedTouches.count)

            for predictedTouch in predictedTouches {
                let locationInView =  predictedTouch.location(in: view)

                predictedDrawPath.move(to: touch.location(in: view))
                predictedDrawPath.addLine(to: locationInView)

                predictedDrawPath.append(UIBezierPath.createCircleAtPoint(origin: locationInView, radius: 1))
            }

            predictedDrawLayer.path = predictedDrawPath.cgPath
        }
        [...]


When you run the app, as you move your finger across the screen, the little predicted touch spots give an indication of where iOS thinks your next touch will be. With a jagged motion, where the prediction doesn’t work so well, you can see those points as little tadpoles following the momentum of your touch.

You can see a few grey circles in the screen grab above which show two touches iOS predicted I would do to complete the spiral - that I never actually did! Spooky!

### Conclusion

As users demand less latency and a higher touch resolution, touch coalescing and prediction allow our apps to support that. Whatever the framerate of our apps, we can respond to all of the user’s gestures and even preempt some of them!

The source code for this demo app is available at my GitHub repository here.
