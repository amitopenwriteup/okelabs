## Verify the repository contents

```bash
oci artifacts container image list \
  --compartment-id $COMPARTMENT_ID \
  --repository-name $REPOSITORY
```

The repository should now contain both `v1` and `v2`.

---

## Task 4: Delete an image by OCID

### Step 1: Identify the image OCID

List the images in the repository and copy the OCID of the image you want to delete:

```bash
oci artifacts container image list \
  --compartment-id "$COMPARTMENT_ID" \
  --repository-name "$REPOSITORY" \
  --query 'data.items[].{
    Image:"display-name",
    ImageOCID:id
  }' \
  --output table
```

### Step 2: Delete the image

Delete the image using its OCID:

```bash
oci artifacts container image delete \
  --image-id <image-ocid>
```

The command normally asks for confirmation. For a lab environment, you can skip it:

```bash
oci artifacts container image delete \
  --image-id <image-ocid> \
  --force
```

> **Warning:** Deleting an image is permanent. Confirm the correct image OCID before running the command.

---

## Task 5: Delete an image by tag using the repository path

Depending on the OCI CLI version, you can identify an image using the repository and image name:

```bash
oci artifacts container image delete \
  --repository-name $REPOSITORY \
  --image-version v1 \
  --compartment-id $COMPARTMENT_ID
```

If this syntax is not accepted by your CLI version, use the image OCID obtained from `container image list`.

---

## Task 6: Delete the repository

To remove the entire repository:

```bash
oci artifacts container repository delete \
  --repository-id $REPOSITORY
```

For non-interactive deletion:

```bash
oci artifacts container repository delete \
  --repository-id $REPOSITORY \
  --force
```
