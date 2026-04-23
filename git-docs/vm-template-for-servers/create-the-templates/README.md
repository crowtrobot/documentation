---
description: >-
  The other pages in this area talk about deploying from the template, so it
  seemed reasonable that there should also be one about the steps to create the
  template.
---

# Create The Templates

We will actually create two templates. They are mostly the same, but one has the root filesystem encrypted with LUKS.&#x20;

The unencrypted template can automatically grow its disk on boot, but we don't want that to happen with the encrypted disk until after it has been re-encrypted with a new unique key, because it will have to re-write the entire filesystem and if the disk was already grown it can more than double the time that process will take. &#x20;
