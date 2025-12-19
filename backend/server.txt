const mongoose = require('mongoose');
const express = require('express');
const cors = require("cors");

const app = express();
app.use(cors());
app.use(express.json());

const MONGO_URI = "mongodb+srv://20225409:20225409@cluster0.vdyr76a.mongodb.net/it4409?appName=Cluster0";
const PORT = 3001;

mongoose.connect(MONGO_URI)
.then(() => {
console.log("Kết nối MongoDB thành công!");
})
.catch((err) => {
console.error("Lỗi kết nối:", err);
});

const UserSchema = new mongoose.Schema({
    name: {
        type: String,
        required: [true, 'Tên không được để trống'],
        minlength: [2, 'Tên phải có ít nhất 2 ký tự']
    },
    age: {
        type: Number,
        required: [true, 'Tuổi không được để trống'],
        min: [0, 'Tuổi phải >= 0']
    },
    email: {
        type: String,
        required: [true, 'Email không được để trống'],
        unique: true,
        match: [/^\S+@\S+\.\S+$/, 'Email không hợp lệ']
    },
    address: {
        type: String
    }
});

const User = mongoose.model('User', UserSchema);

// GET /api/users?page=1&limit=5&search=nguyen
app.get('/api/users', async (req, res) => {
    try {
        const page = parseInt(req.query.page) || 1;
        const limit = parseInt(req.query.limit) || 5;
        const search = req.query.search || "";

        const skip = (page - 1) * limit;

        let query = {};

        if (search) {
            query = {
                $or: [
                    { name: { $regex: search, $options: 'i' } },
                    { email: { $regex: search, $options: 'i' } },
                    { address: { $regex: search, $options: 'i' } }
                ]
            };

        }
        
        const [total, users] = await Promise.all([
            User.countDocuments(query),
            User.find(query).skip(skip).limit(limit)
        ]);
        
        const totalPages = Math.ceil(total / limit);

        res.status(200).json({
            page: page,
            limit: limit,
            total: total,
            totalPages: totalPages,
            data: users
        });
    } catch (error) {
        res.status(500).json({ message: "Lỗi server", error: error.message });
    }
});

// POST /api/users
app.post('/api/users', async (req, res) => {
    try {
        const {name, age, email, address} = req.body;

        const newUser = await User.create({name, age, email, address});

        res.status(201).json({
            message: "Tạo người dùng thành công",
            data: newUser
        });
    } catch (error) {
        if (error.code === 11000) {
            return res.status(400).json({
                error: "Email đã tồn tại. Vui lòng chọn email khác."
            });
        }
        if (error.name === "ValidationError") {
            return res.status(400).json({ error: error.message });
        }
    }
});

app.put("/api/users/:id", async (req, res) => {
    try {
        const {id} = req.params;
        const {name, age, email, address} = req.body;

        const updatedUser = await User.findByIdAndUpdate(
            id,
            {name, age, email, address},
            {new: true, runValidators: true}
        );

        if(!updatedUser) {
            return res.status(404).json({error:"Không tìm thấy người dùng"});
        }

        res.json({
            message: "Cập nhật người dùng thành công",
            data: updatedUser
        })
    } catch (error) {
        if (error.code === 11000) {
            return res.status(400).json({
                error: "Email đã tồn tại. Không thể cập nhật."
            });
        }

        if (error.name === "ValidationError") {
            return res.status(400).json({ error: error.message });
        }
    }
});
app.delete("/api/users/:id", async (req, res) => {
    try {
        const {id} = req.params;
        
        const deletedUser = await User.findByIdAndDelete(id);

        if (!deletedUser) {
            return res.status(404).json({ error: "Không tìm thấy người dùng" });
        }
        res.json({ message: "Xóa người dùng thành công" });
    } catch (err) {
        res.status(400).json({ error: err.message });
    }

});
app.listen(PORT, () => console.log(`Server đang chạy tại http://localhost:${PORT}`));