---
sidebar_position: 4
---

# 4. Handling Upload File With Multer

import useBaseUrl from '@docusaurus/useBaseUrl';

**Multer** merupakan middleware node.js untuk menangani multipart / form-data, yang digunakan untuk upload file, yang nantinya file upload akan disimpan ke dalam folder project, lalu menyetorkan nama dari file tersebut kedalam database

Sebelumnya install terlebih dulu dengan perintah.

```shell
npm install multer
```

selanjutnya import library multer yang sudah di instal
```js title=middlewares/uploadFile.js
const multer = require("multer");
```

configurasi untuk folder penyimpanan file upload dan untuk merubah nama file depannya dengan milisecond waktu sekarang dan digabungkan dengan nama file, dan jika ada spasi maka akan dihapus menjadi tidak ada spasi.

```js title=middlewares/uploadFile.js
exports.uploadFile = (imageFile) => {
    const storage = multer.diskStorage({
        destination: function (req, file, cb) {
        cb(null, "uploads"); //file storage location
        },
        filename: function (req, file, cb) {
        cb(null, Date.now() + "-" + file.originalname.replace(/\s/g, ""));
        },
    });
}
```

Kemudian lakukan configurasi untuk ekstensi atau type filenya agar hanya file berbentuk gambar aja yang bisa di upload.

```js title=middlewares/uploadFile.js
// this code continues from the previous code
const fileFilter = function (req, file, cb) {
    if (file.fieldname === imageFile) {
      if (!file.originalname.match(/\.(jpg|JPG|jpeg|JPEG|png|PNG|gif|GIF)$/)) {
        req.fileValidationError = {
          message: "Only image files are allowed!",
        };
        return cb(new Error("Only image files are allowed!"), false);
      }
    }
    cb(null, true);
};
```

Kemudian tentukan maksimal file yang akan diupload kedalam aplikasi

```js title=middlewares/uploadFile.js
// this code continues from the previous code
const sizeInMB = 10;
  const maxSize = sizeInMB * 1000 * 1000; // Maximum file size in MB
```

kemudian tampung semua configurasi sebelumnya di variabel upload

```js title=middlewares/uploadFile.js
// this code continues from the previous code
const upload = multer({
    storage,
    fileFilter,
    limits: {
      fileSize: maxSize,
    },
  }).single(imageFile);
```
kemudian return dan buat kondisi terkait configurasi sebelumnya,jangan lupa kita kasih next ketika kondisi yang diatas semua terpenuhi, agar dilanjutkan ke controller nantinya.

```js title=middlewares/uploadFile.js
// this code continues from the previous code
return (req, res, next) => {
    upload(req, res, function (err) {
      // show an error if validation failed
      if (req.fileValidationError)
        return res.status(400).send(req.fileValidationError);

      // show an error if file doesn't provided in req
      if (!req.file && !err)
        return res.status(400).send({
          message: "Please select files to upload",
        });

      // show an error if it exceeds the max size
      if (err) {
        if (err.code === "LIMIT_FILE_SIZE") {
          return res.status(400).send({
            message: "Max file sized 10MB",
          });
        }
        return res.status(400).send(err);
      }

      // if okay next to controller
      // in the controller we can access using req.file
      return next();
    });
  };
```


Selanjutnya kita akan mencoba implementasikan, berikut contoh codenya:

<a class="btn-example-code" href="https://github.com/demo-dumbways/ebook-code-results-stage-2-backend/blob/1-expressjs-fundamental/index.js">
Contoh code
</a>

<br />
<br />

```js title=middlewares/uploadFile.js
const multer = require("multer");

exports.uploadFile = (imageFile) => {
  // initialization multer diskstorage
  // make destination file for upload
  const storage = multer.diskStorage({
    destination: function (req, file, cb) {
      cb(null, "uploads"); //file storage location
    },
    filename: function (req, file, cb) {
      cb(null, Date.now() + "-" + file.originalname.replace(/\s/g, "")); // rename filename by date now + original filename
    },
  });

  // function for file filter based on extension
  const fileFilter = function (req, file, cb) {
    if (file.fieldname === imageFile) {
      if (!file.originalname.match(/\.(jpg|JPG|jpeg|JPEG|png|PNG|gif|GIF)$/)) {
        req.fileValidationError = {
          message: "Only image files are allowed!",
        };
        return cb(new Error("Only image files are allowed!"), false);
      }
    }
    cb(null, true);
  };

  const sizeInMB = 10;
  const maxSize = sizeInMB * 1000 * 1000; // Maximum file size in MB

  // generate multer instance for upload include storage, validation and max file size
  const upload = multer({
    storage,
    fileFilter,
    limits: {
      fileSize: maxSize,
    },
  }).single(imageFile);

  // middleware handler
  return (req, res, next) => {
    upload(req, res, function (err) {
      // show an error if validation failed
      if (req.fileValidationError)
        return res.status(400).send(req.fileValidationError);

      // show an error if file doesn't provided in req
      if (!req.file && !err)
        return res.status(400).send({
          message: "Please select files to upload",
        });

      // show an error if it exceeds the max size
      if (err) {
        if (err.code === "LIMIT_FILE_SIZE") {
          return res.status(400).send({
            message: "Max file sized 10MB",
          });
        }
        return res.status(400).send(err);
      }

      // if okay next to controller
      // in the controller we can access using req.file
      return next();
    });
  };
};
```

```js title=controllers/product.js {51,108}
const { product, user, category, productCategory } = require("../../models");

exports.getProduct = async (req, res) => {
  try {
    const data = await product.findAll({
      include: [
        {
          model: user,
          as: "user",
          attributes: {
            exclude: ["createdAt", "updatedAt", "password"],
          },
        },
        {
          model: category,
          as: "categories",
          through: {
            model: productCategory,
            as: "bridge",
            attributes: [],
          },
          attributes: {
            exclude: ["createdAt", "updatedAt"],
          },
        },
      ],
      attributes: {
        exclude: ["createdAt", "updatedAt", "idUser"],
      },
    });

    res.send({
      status: "success...",
      data,
    });
  } catch (error) {
    console.log(error);
    res.send({
      status: "failed",
      message: "Server Error",
    });
  }
};

exports.addProduct = async (req, res) => {
  try {
    const { category: categoryName, ...data } = req.body;

    const newProduct = await product.create({
      ...data,
      image: req.file.filename, // image that passed from middleware will in the req.file 
      idUser: req.user.id, // got from authentication middleware

    });
    const categoryData = await category.findOne({
      where: {
        name: categoryName,
      },
    });

    if (categoryData) {
      await productCategory.create({
        idCategory: categoryData.id,
        idProduct: newProduct.id,
      });
    } else {
      const newCategory = await category.create({ name: categoryName });
      await productCategory.create({
        idCategory: newCategory.id,
        idProduct: newProduct.id,
      });
    }
    let productData = await product.findOne({
      where: {
        id: newProduct.id,
      },
      include: [
        {
          model: user,
          as: "user",
          attributes: {
            exclude: ["createdAt", "updatedAt", "password"],
          },
        },
        {
          model: category,
          as: "categories",
          through: {
              model: productCategory,
            as: "bridge",
            attributes: [],
          },
          attributes: {
            exclude: ["createdAt", "updatedAt"],
          },
        },
      ],
      attributes: {
        exclude: ["createdAt", "updatedAt", "idUser"],
      },
    });
    productData = JSON.parse(JSON.stringify(productData))

    res.send({
      status: "success...",
      data: {
        ...productData,
        image: 'http://localhost:5000/uploads/' + productData.image
      },
    });
  } catch (error) {
    console.log(error);
    res.status(500).send({
      status: "failed",
      message: "Server Error",
    });
  }
};
```

```js title=routes/index.js {13,23}
const express = require('express')

const router = express.Router()

// Controller
const { addUsers, getUsers, getUser, updateUser, deleteUser } = require('../controllers/user')
const { getProduct, addProduct } = require('../controllers/product')
const { getTransactions, addTransaction } = require('../controllers/transaction')
const { register, login } = require('../controllers/auth')

// Middleware
const { auth } = require('../middlewares/auth')
const { uploadFile } = require('../middlewares/uploadFile')

// Route
router.post('/user', addUsers)
router.get('/users', getUsers)
router.get('/user/:id', getUser)
router.patch('/user/:id', updateUser)
router.delete('/user/:id', deleteUser)

router.get('/products', getProduct)
router.post('/product', auth, uploadFile("image"), addProduct)

router.get('/transactions', getTransactions)
router.post('/transaction', auth, addTransaction)

router.post('/register', register)
router.post('/login', login)

module.exports = router
```

Selanjutnya kita buat path di index.js agar gambarnya bisa tampil atau bisa diakses secara public
```js {16}
// import dotenv and call config function to load environment
require('dotenv').config()
const express = require('express')

// Get routes to the variabel
const router = require('./src/routes')

const app = express()

const port = 5000

app.use(express.json())

// Add endpoint grouping and router
app.use('/api/v1/', router)
app.use('/uploads', express.static('uploads'))

app.listen(port, () => console.log(`Listening on port ${port}!`))
```

<img alt="image1-2" src={useBaseUrl('img/docs/image-4-1.png')} width="60%"/>

<br />
<br />

<div>
<a class="btn-demo" href="https://ebook-code-results-stage-2-backend-git-1-e-bef277-demo-dumbways.vercel.app/">
Demo
</a>
</div>