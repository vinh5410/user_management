const express = require("express");
const mongoose = require("mongoose");
const cors = require("cors");
const app = express();

// Middleware
app.use(cors());
app.use(express.json());

// Kết nối MongoDB 
mongoose
  .connect("mongodb+srv://20225238:20225238@cluster0.bwpkzkf.mongodb.net/it4409?retryWrites=true&w=majority", {
  })
  .then(() => console.log("Connected to MongoDB"))
  .catch((err) => console.error("MongoDB Error:", err));

// User Schema
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
    unique: true,
    match: [/^\S+@\S+\.\S+$/, 'Email không hợp lệ'],
    lowercase: true,
    trim: true
  },
  address: {
    type: String,
  }
}, {
  timestamps: true // Tự động thêm createdAt, updatedAt
});

// Thêm index để tăng tốc search
UserSchema.index({ name: 'text', email: 'text', address: 'text' });

const User = mongoose.model("User", UserSchema);

// GET All Users với pagination và search
app.get("/api/users", async (req, res) => {
  try {
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 5;
    const search = req.query.search || "";

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

    // Sử dụng Promise.all để chạy song song
    const [users, total] = await Promise.all([
      User.find(filter).skip(skip).limit(limit).lean(), // .lean() để tăng tốc
      User.countDocuments(filter)
    ]);

    const totalPages = Math.ceil(total / limit);

    res.json({
      success: true,
      page,
      limit,
      total,
      totalPages,
      data: users
    });
  } catch (err) {
    console.error("GET Error:", err);
    res.status(500).json({ 
      success: false,
      error: err.message 
    });
  }
});

// POST Create User
app.post("/api/users", async (req, res) => {
  try {
    const { name, age, email, address } = req.body;
    if(age&&!Number.isInteger(Number(age))){
        return res.status(400).json({ 
        success: false,
        error: "Tuổi phải là số nguyên"
      });
    }
    const existingUser = await User.findOne({ email: email.trim().toLowerCase() });
    if (existingUser) {
      return res.status(400).json({
        success: false,
        error: "Email đã được sử dụng"
      });
    }
    const newUser = await User.create({ name, age, email, address });

    res.status(201).json({
      success: true,
      message: "Tạo người dùng thành công",
      data: newUser
    });
  } catch (err) {
    console.error("POST Error:", err);
    
    if (err.name === 'ValidationError') {
      return res.status(400).json({
        success: false,
        error: Object.values(err.errors).map(e => e.message).join(', ')
      });
    }
    
    res.status(400).json({ 
      success: false,
      error: err.message 
    });
  }
});

// PUT Update User
app.put("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;
    const { name, age, email, address } = req.body;

    const updatedUser = await User.findByIdAndUpdate(
      id,
      { name, age, email, address },
      { new: true, runValidators: true }
    );

    if (!updatedUser) {
      return res.status(404).json({ 
        success: false,
        error: "Không tìm thấy người dùng" 
      });
    }

    res.json({
      success: true,
      message: "Cập nhật người dùng thành công",
      data: updatedUser
    });
  } catch (err) {
    console.error("PUT Error:", err);
    
    if (err.kind === 'ObjectId') {
      return res.status(400).json({
        success: false,
        error: "ID không hợp lệ"
      });
    }
    
    res.status(400).json({ 
      success: false,
      error: err.message 
    });
  }
});

// DELETE User
app.delete("/api/users/:id", async (req, res) => {
  try {
    const { id } = req.params;

    const deletedUser = await User.findByIdAndDelete(id);

    if (!deletedUser) {
      return res.status(404).json({ 
        success: false,
        error: "Không tìm thấy người dùng" 
      });
    }

    res.json({ 
      success: true,
      message: "Xóa người dùng thành công" 
    });
  } catch (err) {
    console.error("DELETE Error:", err);
    
    if (err.kind === 'ObjectId') {
      return res.status(400).json({
        success: false,
        error: "ID không hợp lệ"
      });
    }
    
    res.status(400).json({ 
      success: false,
      error: err.message 
    });
  }
});

// Home route
app.get("/", (req, res) => {
  res.json({
    message: "User Management API",
    endpoints: {
      "GET /api/users": "Lấy danh sách users (có pagination & search)",
      "POST /api/users": "Tạo user mới",
      "PUT /api/users/:id": "Cập nhật user",
      "DELETE /api/users/:id": "Xóa user"
    }
  });
});

// Start server
app.listen(3001, () => {
  console.log("Server running on http://localhost:3001");
});