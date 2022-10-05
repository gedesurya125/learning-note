# Scroll Smoother With Gatsby

1. Create a hook like code bellow. Note: we should keep in mind that smoothScroller will be set up globally at layout index, and because of smoothScroller is created on ScrollTrigger, it means we use same ScrollTrigger everywhere between pages

```js
// scrollSmootherContext.js

import React from "react";
import { gsap } from "gsap";
import { ScrollTrigger } from "gsap/ScrollTrigger";
import { ScrollSmoother } from "gsap/ScrollSmoother";
import { Box } from "theme-ui";
import { useLocation } from "@reach/router";

gsap.registerPlugin(ScrollTrigger, ScrollSmoother);

export const SmootherContext = React.createContext({ scroller: null });

export const useScrollSmoother = () => {
  return React.useContext(SmootherContext);
};

export const SmootherProvider = ({ children }) => {
  const { pathname } = useLocation();
  const [value, setValue] = React.useState({
    // latter scroller will be used to indicate if the smoothScroller is ready then create the animation
    scroller: null,
  });

  React.useEffect(() => {
    const scroller = ScrollSmoother.create({
      wrapper: "#smooth-wrapper",
      content: "#smooth-content",
      smooth: 1.5,
    });
    setValue((state) => ({ ...state, scroller }));

    // Detect document height change due to font size change or image lazy loaded
    const resizeObserver = new ResizeObserver(() => scroller.refresh());
    resizeObserver.observe(document.body);

    return () => {
      // important to make the navbar animation (that use hide on scroll down) not break. It make the page scrolled to the top before it is refresed. false means don't animate the scrolling
      scroller.scrollTo(0, false);
      // refresth the scroller and the scroll triggered animation inside, we use refresh instead of kill because it will used in layout component , if we kill, the scroller , it sill no longer exits at all pages
      scroller.refresh();
    };
    // We need to recreate the scroll smoother each page change, to make the scroll trigger have a new reference, if not the old scrollSmoother properties will be applied to new page for example, page height, and ScrollTrigger markers
  }, [pathname]);

  return (
    <SmootherContext.Provider value={value}>
      <Box id="smooth-wrapper">
        <Box id="smooth-content">{children}</Box>
      </Box>
    </SmootherContext.Provider>
  );
};
```

2. Apply it to the layout component

```js
// layout/index.js

<>
  {/* Other wrapper goes here*/}
  {/* Navigation bar need to be outside as it's Fixed positioned*/}
  <Navigation />
  <SmootherProvider>
    <Box as="main">{/* Content Goes Here */}</Box>
    <Footer />
  </SmootherProvider>
  {/* Other closing wrapper goes here*/}
</>
```

3. We have to make sure that the scroller of scrollSmoother is set up before we create the animation, to ensure that the animation can run properly

```js
// For example Header.js
export const Header = (props) => {
  // get the created scroller using the hook that we created at the first step
  const { scroller } = useScrollSmoother();


  React.useEffect(() => {
    // Create the aimation only if the scroller is set up
    // if not the animation will not run properly
    if (scroller === null) return;
    const killAnimation = createAnimation();
    return killAnimation;

  }, [scroller]);

  return (
    // Markup goes here
  )
}
```

4. For the animation that use `ScrollTrigger.matchMedia({})` we have to kill the animation that created inside the media query mannualy, because, it's not killed automatically by the ScrollTrigger when the page change, so we can do something like this:

```js
export const createAnimation = () => {
  // Elements
  const elementToAnimate = document.querySelector(".element-to-animate");

  //Sometimes it's required, to fix issue with scrollSmotther that when we back to this page the element would not use it's last animated position as a initial value
  ScrollTrigger.saveStyles(elementToAnimate);

  // Create tl variable to store the animation, so we can kill it manually latter
  // because scrollTrigger inside the match media not killed properlly as what should be done by the ScrollTrigger.matchMedia
  let tl;
  ScrollTrigger.matchMedia({
    // Example of breakpoints
    "(min-width: 1000px)": function () {
      tl = gsap.timeline({
        scrollTrigger: {
          trigger: imageWrapper,
          start: `top ${triggerStart}`,
          end: `80% ${triggerStart}`,
          scrub: 0,
        },
      });

      tl.to(elementToAnimate, {
        y: 1000,
      });

      return () => {
        tl.kill();
        tl?.scrollTrigger?.kill();
      };
    },
  });

  // return the killer to the useEffect
  return () => {
    if (tl) {
      tl.kill();
      // it is important to make the scroll trigger killed
      tl?.scrollTrigger?.kill();
    }
  };
};
```
