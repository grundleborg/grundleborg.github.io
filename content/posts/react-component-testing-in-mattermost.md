+++
date = "2017-05-02T16:25:32+01:00"
draft = false
title = "React Component Testing in Mattermost"
+++

[Mattermost][1] is a large web app written in [ReactJS][2]. It currently consists of a little over 300 components, built using a "Flux-like" architecture. This makes unit testing of components particularly challenging as the vast majority of them call methods on the stores directly. Of course, it's *possible* to mock out the stores in every
  test, but it's not really *practical* to do this.
  
If that was the end of the story, it would be a very boring one indeed. Fortunately, we are in the process of changing the underlying architecture of the Mattermost web app. After testing out [Redux][3] as the core of our new [React Native][4] powered [mobile apps][5], we've decided to move over to the same architecture in our web app. I'll leave writing about the details of how that migration is happening to someone else, and instead focus on the impact of this on component testing.

As part of the transition to Redux, our component structure is changing. Instead of accessing stores from any component whenever some store data is needed, components will now be logically grouped into hierarchies of pure, UI components, receiving all their data through their `props`, with store access limited to a single "controller" UI-less component acting as the parent to each group of UI components.

This change of architecture brings a major benefit for component testing: UI components no longer have complicated side effects, extensive dependency chains, or direct access to data sources. Instead, there is a single API onto each component, described by its `props` field. This makes unit testing each component in isolation a trivial task.

From the point of view of the application as a whole, having well defined API contracts that are extensively unit tested eases the reuse of components (and reduces the appeal of copy-paste code blocks being littered throughout the code base  - something that currently happens rather more often than we'd ideally like). It also provides a structure and set of expectations around the behaviour of components, which reduces the need to examine the internals of every child component (and grandchild...ad infinitum) to investigate bugs or make simple feature changes to a component, as is often the case in our current code base.

## Jest and Enzyme

Mattermost uses [Jest][6] to run the web app tests, in combination with [Enzyme][7]. The combination of these two utilities gives us a very simple workflow for component testing. Enzyme allows us to render and interact with React components in isolation, and Jest allows us to perform snapshot testing against these components - allowing us to detect any changes in rendered output of components and, where such a change is intended, automatically update the test snapshots. The integration between these two utilities is provided by the [enzyme-to-json][8] library.

I won't get into the details of how we set up the plumbing for these libraries - plenty of other people have written about it (e.g. [here][9]), and the only differences between what they and I have done are specific to the Mattermost codebase. If you are interested, you can have a look through the [Mattermost web app][10] source code.

## The Tests

To recap for a moment: the purpose of the component tests we are building is to verify the API contract of the UI components, expressed in terms of their `props`. However, since we don't yet live in a perfect world of 100% pure components, where practical we also want to test any known side-effects of global state on them.

Let's consider two example components. The first is the `BackstageHeader` component (full source code [here][11]), chosen for its extreme simplicity, and the fact it is already a pure component. The render method is shown here:

~~~javascript
render() {
    const children = [];

    React.Children.forEach(this.props.children, (child, index) => {
        if (index !== 0) {
            children.push(
                <span
                    key={'divider' + index}
                    className='backstage-header__divider'
                >
                    <i className='fa fa-angle-right'/>
                </span>
            );
        }

        children.push(child);
    });

    return (
        <div className='backstage-header'>
            <h1>
                {children}
            </h1>
        </div>
    );
}
~~~

And the props:

~~~javascript
static get propTypes() {
    return {
        children: React.PropTypes.node
    };
}
~~~

This is a very simple component - it just builds a header based on the single, optional `children` prop, containing a list of child objects to insert into the header.

### Shallow Rendering

The component tests we are building are *unit* tests, so we would like to limit the testing to individual components, and avoid changes in components elsewhere in the hierarchy affecting these tests. Enzyme provides a very useful tool for this: shallow rendering. Shallow rendering allows us to render just this component, but not any of it's child components, allowing it to be tested entirely in isolation.
  
~~~javascript
test('should match snapshot without children', () => {
    const wrapper = shallow(
        <BackstageHeader/>
    );
    expect(wrapper).toMatchSnapshot();
});
~~~

This first, very simple test shallow-renders the `BackstageHeader` component, and makes use of Jest's built in snapshot testing capability to compare the rendered component with a known-good snapshot (more on that in the next section). As the `children` prop is optional, we test the component initially without any children to ensure it behaves properly.

The only other test we need here is one *with* the `children` prop set: 

~~~javascript
test('should match snapshot with children', () => {
    const wrapper = shallow(
        <BackstageHeader>
            <div>{'Child 1'}</div>
            <div>{'Child 2'}</div>
        </BackstageHeader>
    );
    expect(wrapper).toMatchSnapshot();
});
~~~

That's it for this very simple component. You can see the `BackstageHeader` test suite in full [here][12].
  
### Running Tests
If you are adding or modifying existing component tests, you can run jest in "watch" mode like this:

~~~bash
user@host webapp$ npm run test:watch
~~~

This will run all tests that have been modified, and then continue to watch for further modifications, re-running tests as necessary. On the first run with new tests, it'll generate the test snapshots, placing them in the `__snapshots__` directory inside the directory where the test suite is located. You'll then see output like this:

~~~
No tests found related to files changed since last commit.
Press `a` to run all tests, or run Jest with `--watchAll`.

Watch Usage
 › Press a to run all tests.
 › Press p to filter by a filename regex pattern.
 › Press t to filter by a test name regex pattern.
 › Press q to quit watch mode.
 › Press Enter to trigger a test run.

~~~

For the `BackstageHeader` test, let's take a look at the snapshot that has been generated:

~~~javascript
// Jest Snapshot v1, https://goo.gl/fbAQLP

exports[`components/backstage/components/BackstageHeader should match snapshot with children 1`] = `
<div
  className="backstage-header"
>
  <h1>
    <div>
      Child 1
    </div>
    <span
      className="backstage-header__divider"
    >
      <i
        className="fa fa-angle-right"
      />
    </span>
    <div>
      Child 2
    </div>
  </h1>
</div>
`;

exports[`components/backstage/components/BackstageHeader should match snapshot without children 1`] = `
<div
  className="backstage-header"
>
  <h1 />
</div>
`;
~~~

We can see that this contains the rendered output for both test cases shown above. You won't ever need to manually edit this file - Jest will take care of that for you, as we are about to find out.

Let's make a change to the `BackstageHeader` component. Instead of using the FontAwesome right-angle-bracket between each header item, let's use the left-angle-bracket instead. We change the relevant line in `backstage_header.jsx` from `<i className='fa fa-angle-right'/>` to `<i className='fa fa-angle-left'/>`. Switching back to the shell where we ran the tests in watch mode previously, we see the following output:

~~~
 FAIL  tests/components/backstage/components/backstage_header.test.jsx
  ● components/backstage/components/BackstageHeader › should match snapshot with children

    expect(value).toMatchSnapshot()
    
    Received value does not match stored snapshot 1.
    
    - Snapshot
    + Received
    
    @@ -7,11 +7,11 @@
         </div>
         <span
           className="backstage-header__divider"
         >
           <i
    -        className="fa fa-angle-right"
    +        className="fa fa-angle-left"
           />
         </span>
         <div>
           Child 2
         </div>
      
      at Object.<anonymous> (tests/components/backstage/components/backstage_header.test.jsx:24:25)
      at process._tickCallback (internal/process/next_tick.js:109:7)

  components/backstage/components/BackstageHeader
    ✓ should match snapshot without children (3ms)
    ✕ should match snapshot with children (5ms)

Snapshot Summary
 › 1 snapshot test failed in 1 test suite. Inspect your code changes or press `u` to update them.

Test Suites: 1 failed, 1 total
Tests:       1 failed, 1 passed, 2 total
Snapshots:   1 failed, 1 passed, 2 total
Time:        1.896s
Ran all test suites related to changed files.

Watch Usage
 › Press a to run all tests.
 › Press u to update failing snapshots.
 › Press p to filter by a filename regex pattern.
 › Press t to filter by a test name regex pattern.
 › Press q to quit watch mode.
 › Press Enter to trigger a test run.
~~~

In that output, you can see that one of the test cases has passed, but the other has failed, due to the change to the symbol displayed between the child components. Helpfully, a diff of the rendered output is displayed. By looking at the diff, we can see that the test case failing is due to our intentional change to `BackstageHeader`, so we should update the test snapshot, rather than fix a bug which the tests have brought to light. We can do this simply by pressing the letter `u`. Once this has completed, we see the following output indicating that the tests are now passing once more:

~~~
PASS  tests/components/backstage/components/backstage_header.test.jsx
  components/backstage/components/BackstageHeader
    ✓ should match snapshot without children (4ms)
    ✓ should match snapshot with children (3ms)

Snapshot Summary
 › 1 snapshot updated in 1 test suite.

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   1 updated, 1 passed, 2 total
Time:        0.528s, estimated 2s
Ran all test suites related to changed files.

Watch Usage
 › Press a to run all tests.
 › Press p to filter by a filename regex pattern.
 › Press t to filter by a test name regex pattern.
 › Press q to quit watch mode.
 › Press Enter to trigger a test run.
 ~~~

The contents of the `__snapshots__` directories are an essential part of the test suite. Be sure to always run the full test suite to generate them after adding or modifying any components or test code, and to include them in your git commits.

### Global State
The `BackstageHeader` component is a very simple, pure component. But how about a more complex component? For this second example, we will use the `AboutBuildModal` component. The source code for this component is too long to reproduce in full, but can be seen [here][13]. The props it expects are as follows:

~~~javascript
AboutBuildModal.propTypes = {
    show: React.PropTypes.bool.isRequired,
    onModalDismissed: React.PropTypes.func.isRequired
};
~~~

Testing how it renders with `show` set to `true` and `false` respectively would be simple enough, but by examining the source code of the component, we see that it also renders differently depending on some global state. In order to test this, we need test cases covering the full range of global state which can have an effect on how it renders. One example is shown below:
 
~~~javascript
test('should show the build number if it is the different from the version number', () => {
    global.window.mm_config = {
        BuildEnterpriseReady: 'false',
        Version: '3.6.0',
        BuildNumber: '3.6.2',
        SQLDriverName: 'Postgres',
        BuildHash: 'abcdef1234567890',
        BuildDate: '21 January 2017'
    };

    global.window.mm_license = null;

    const wrapper = shallow(
        <AboutBuildModal
            show={true}
            onModalDismissed={null}
        />
    );
    expect(wrapper.find('#versionString').text()).toBe(' 3.6.0\u00a0 (3.6.2)');
});
~~~

That's just one of the test cases needed to ensure the different ways in which the component can render are all covered. The full test suite for the `AboutBuildModal` can be seen [here][14].

### Mounting Components
 
Continuing with the `AboutBuildModal`, there's one other prop we want to test: the `onModalDismissed` callback. This should be triggered when the `Modal` component within the `AboutBuildModal` is hidden. In order to interact with the full component hierarchy within the `AboutBuildModal`, we need to do a full render, or "mount", rather than a shallow render as we've been using previously. We can then use the `find()` method on the wrapper to locate the inner component and trigger it as if it has been hidden by the user, to test if our `onModalDismissed` prop is correctly called.

~~~javascript
test('should call onModalDismissed callback when the modal is hidden', (done) => {
    global.window.mm_config = {
        BuildEnterpriseReady: 'false',
        Version: '3.6.0',
        BuildNumber: '3.6.2',
        SQLDriverName: 'Postgres',
        BuildHash: 'abcdef1234567890',
        BuildDate: '21 January 2017'
    };

    global.window.mm_license = null;

    function onHide() {
        done();
    }

    const wrapper = mountWithIntl(
        <AboutBuildModal
            show={true}
            onModalDismissed={onHide}
        />
    );
    wrapper.find(Modal).first().props().onHide();
});
~~~

### React i18n Test Helper
If you were looking closely, you might have noticed that we used a function called `mountWithIntl()` in the previous test case. This is a custom function that wraps the `mount()` function provided by enzyme, in order to inject the props needed for React i18n support to work. Whenever you are shallow rendering or mounting a component which will result in the rendering of localised strings, you should use the appropriate wrapper functions to ensure they work correctly. The code is very simple and shown below (you can see it in the Mattermost source tree [here][15]). 

~~~javascript
import {mount, shallow} from 'enzyme';
import React from 'react';
import {IntlProvider, intlShape} from 'react-intl';

const intlProvider = new IntlProvider({locale: 'en'}, {});
const {intl} = intlProvider.getChildContext();

export function shallowWithIntl(node, {context} = {}) {
    return shallow(React.cloneElement(node, {intl}), {
        context: Object.assign({}, context, {intl})
    });
}

export function mountWithIntl(node, {context, childContextTypes} = {}) {
    return mount(React.cloneElement(node, {intl}), {
        context: Object.assign({}, context, {intl}),
        childContextTypes: Object.assign({}, {intl: intlShape}, childContextTypes)
    });
}
~~~

### Testing events
There's one other major feature of enzyme that we haven't shown in these examples: event simulation. Enzyme allows you to simulate events in order to test your event handlers. This can be done by finding the relevant item on the rendered `wrapper` object, then using the `simulate()` method to trigger an event, and using `toBeCalled()` to assert that the appropriate handler is called. The previous example test could thus be rewritten as follows:

~~~javascript
test('should call onModalDismissed callback when the modal is hidden', () => {
    global.window.mm_config = {
        BuildEnterpriseReady: 'false',
        Version: '3.6.0',
        BuildNumber: '3.6.2',
        SQLDriverName: 'Postgres',
        BuildHash: 'abcdef1234567890',
        BuildDate: '21 January 2017'
    };

    const onHide = jest.fn();
    const wrapper = mountWithIntl(
        <AboutBuildModal
            show={true}
            onModalDismissed={onHide}
        />
    );
    wrapper.find(Modal).first().simulate('hide');
    expect(onHide).toBeCalled();
});
~~~

## Conclusion
Component tests provide an easy way of unit testing React components, ensuring consistency in their rendered output, and enforcing a reliable, props-based API. As we move the web app codebase to Redux, the potential benefits increase dramatically, as almost all UI components will be fully testable in this way. Going forward, we expect to make component tests a requirement for all new (or refactored) UI components introduced into the Mattermost webapp in the near future, similar to how we already require test coverage for all server-side changes. You can keep see the Mattermost webapp's component tests as they are added [here][16].

[1]: https://github.com/mattermost/platform/
[2]: https://facebook.github.io/react/
[3]: http://redux.js.org/docs/introduction/
[4]: https://facebook.github.io/react-native/
[5]: https://github.com/mattermost/mattermost-mobile
[6]: https://facebook.github.io/jest/
[7]: http://airbnb.io/enzyme/
[8]: https://www.npmjs.com/package/enzyme-to-json
[9]: https://hackernoon.com/testing-react-components-with-jest-and-enzyme-41d592c174f
[10]: https://github.com/mattermost/platform/tree/master/webapp
[11]: https://github.com/mattermost/platform/blob/v3.8.2/webapp/components/backstage/components/backstage_header.jsx
[12]: https://github.com/mattermost/platform/blob/v3.8.2/webapp/tests/components/backstage/components/backstage_header.test.jsx
[13]: https://github.com/mattermost/platform/blob/v3.8.2/webapp/components/about_build_modal.jsx
[14]: https://github.com/mattermost/platform/blob/v3.8.2/webapp/tests/components/about_build_modal.test.jsx
[15]: https://github.com/mattermost/platform/blob/v3.8.2/webapp/tests/helpers/intl-test-helper.jsx
[16]: https://github.com/mattermost/platform/tree/master/webapp/tests/components

