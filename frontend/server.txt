const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");

const app = express();

app.use(cors());
app.use(express.json());

const MONGODB_URI = "mongodb://20235401:Pt06041507@ac-gw9vjr7-shard-00-00.tzwhm8x.mongodb.net:27017,ac-gw9vjr7-shard-00-01.tzwhm8x.mongodb.net:27017,ac-gw9vjr7-shard-00-02.tzwhm8x.mongodb.net:27017/it4409?ssl=true&replicaSet=atlas-6aert1-shard-0&authSource=admin&retryWrites=true&w=majority&appName=it4409-20235401";

mongoose
  .connect(MONGODB_URI)
  .then(() => console.log("Connected to MongoDB Atlas"))
  .catch((err) => console.error("MongoDB Error:", err));

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Tên không được để trống'],
    minlength: [2, 'Tên phải có ít nhất 2 ký tự'],
    trim: true
  },
  age: {
    type: Number,
    required: [true, 'Tuổi không được để trống'],
    min: [0, 'Tuổi phải >= 0'],
    validate: {
      validator: Number.isInteger,
      message: 'Tuổi phải là số nguyên'
    }
  },
  email: {
    type: String,
    required: [true, 'Email không được để trống'],
    match: [/^\S+@\S+\.\S+$/, 'Email không hợp lệ'],
    trim: true,
    unique: true
  },
  address: {
    type: String,
    trim: true
  }
});

const User = mongoose.model("User", UserSchema);

const formatError = (err) => {
  if (err.code === 11000) {
    return "Email đã tồn tại";
  }
  if (err.name === 'ValidationError') {
    return Object.values(err.errors).map(val => val.message).join(', ');
  }
  return err.message;
};

app.get("/api/users", async (req, res) => {
  try {
    let page = parseInt(req.query.page) || 1;
    let limit = parseInt(req.query.limit) || 5;

    if (page < 1) page = 1;
    if (limit < 1) limit = 1;
    if (limit > 100) limit = 100;

    const search = req.query.search ? req.query.search.trim() : "";

    const filter = search
      ? {
        $or: [
          { name: { $regex: search, $options: "i" } },
          { email: { $regex: search, $options: "i" } },
          { address: { $regex: search, $options: "i" } }
        ]
      }
      : {};

    const skip = (page - 1) * limit;

    const [users, total] = await Promise.all([
      User.find(filter).skip(skip).limit(limit),
      User.countDocuments(filter)
    ]);

    const totalPages = Math.ceil(total / limit);

    res.json({
      page,
      limit,
      total,
      totalPages,
      data: users
    });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.post("/api/users", async (req, res) => {
  try {
    const { name, age, email, address } = req.body;

    const newUser = await User.create({ name, age, email, address });

    res.status(201).json({
      message: "Tạo người dùng thành công",
      data: newUser
    });
  } catch (err) {
    res.status(400).json({ error: formatError(err) });
  }
});

app.put("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;

    if (!mongoose.Types.ObjectId.isValid(id)) {
      return res.status(400).json({ error: "ID không hợp lệ" });
    }

    const { name, age, email, address } = req.body;

    const updateData = {};
    if (name !== undefined) updateData.name = name;
    if (age !== undefined) updateData.age = age;
    if (email !== undefined) updateData.email = email;
    if (address !== undefined) updateData.address = address;

    const updatedUser = await User.findByIdAndUpdate(
      id,
      updateData,
      { new: true, runValidators: true }
    );

    if (!updatedUser) {
      return res.status(404).json({ error: "Không tìm thấy người dùng" });
    }

    res.json({
      message: "Cập nhật người dùng thành công",
      data: updatedUser
    });
  } catch (err) {
    res.status(400).json({ error: formatError(err) });
  }
});

app.delete("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;

    if (!mongoose.Types.ObjectId.isValid(id)) {
      return res.status(400).json({ error: "ID không hợp lệ" });
    }

    const deletedUser = await User.findByIdAndDelete(id);

    if (!deletedUser) {
      return res.status(404).json({ error: "Không tìm thấy người dùng" });
    }

    res.json({ message: "Xóa người dùng thành công" });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

app.listen(3001, () => {
  console.log("Server running on http://localhost:3001");
});
