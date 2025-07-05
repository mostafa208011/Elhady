import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:image_picker/image_picker.dart';
import 'dart:io';
import 'package:intl/intl.dart';
import 'package:flutter_typeahead/flutter_typeahead.dart';

void main() {
  runApp(LivestockApp());
}

class LivestockApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'إدارة الماشية',
      theme: ThemeData(
        primarySwatch: Colors.green,
        fontFamily: 'Cairo',
        appBarTheme: AppBarTheme(
          backgroundColor: Colors.green[700],
          titleTextStyle: TextStyle(
            fontSize: 22,
            fontWeight: FontWeight.bold,
            color: Colors.white,
          ),
        ),
      ),
      home: HomePage(),
      debugShowCheckedModeBanner: false,
    );
  }
}

class HomePage extends StatefulWidget {
  @override
  _HomePageState createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  int _currentIndex = 0;

  final List<Widget> _pages = [
    AnimalsListPage(),
    AddAnimalPage(),
    VaccinationCalendarPage(),
  ];

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('إدارة الماشية'),
        centerTitle: true,
        actions: [
          if (_currentIndex == 0)
            IconButton(
              icon: Icon(Icons.search),
              onPressed: () {
                showSearch(context: context, delegate: AnimalSearchDelegate());
              },
            ),
        ],
      ),
      body: _pages[_currentIndex],
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _currentIndex,
        onTap: (index) {
          setState(() {
            _currentIndex = index;
          });
        },
        items: [
          BottomNavigationBarItem(
            icon: Icon(Icons.list),
            label: 'قائمة الحيوانات',
          ),
          BottomNavigationBarItem(
            icon: Icon(Icons.add),
            label: 'إضافة حيوان',
          ),
          BottomNavigationBarItem(
            icon: Icon(Icons.calendar_today),
            label: 'التطعيمات',
          ),
        ],
        backgroundColor: Colors.green[700],
        selectedItemColor: Colors.white,
        unselectedItemColor: Colors.white70,
      ),
    );
  }
}

class AnimalsListPage extends StatefulWidget {
  @override
  _AnimalsListPageState createState() => _AnimalsListPageState();
}

class _AnimalsListPageState extends State<AnimalsListPage> {
  final TextEditingController _searchController = TextEditingController();
  String _searchTerm = '';

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Filter Row
            Row(
              children: [
                Expanded(
                  child: Container(
                    decoration: BoxDecoration(
                      color: Colors.white,
                      borderRadius: BorderRadius.circular(30),
                      boxShadow: [
                        BoxShadow(
                          color: Colors.black12,
                          blurRadius: 5,
                          offset: Offset(0, 3),
                        ),
                      ],
                    ),
                    child: TextField(
                      controller: _searchController,
                      onChanged: (value) {
                        setState(() {
                          _searchTerm = value.toLowerCase();
                        });
                      },
                      decoration: InputDecoration(
                        hintText: 'ابحث باسم الحيوان...',
                        prefixIcon: Icon(Icons.search),
                        border: InputBorder.none,
                        contentPadding: EdgeInsets.symmetric(horizontal: 20, vertical: 15),
                      ),
                    ),
                  ),
                ),
                SizedBox(width: 10),
                Container(
                  decoration: BoxDecoration(
                    color: Colors.green,
                    borderRadius: BorderRadius.circular(12),
                  ),
                  child: IconButton(
                    icon: Icon(Icons.filter_list, color: Colors.white),
                    onPressed: () {
                      _showFilterDialog(context);
                    },
                  ),
                ),
              ],
            ),
            SizedBox(height: 20),
            
            Expanded(
              child: StreamBuilder<QuerySnapshot>(
                stream: FirebaseFirestore.instance.collection('الحيوانات').snapshots(),
                builder: (context, snapshot) {
                  if (snapshot.connectionState == ConnectionState.waiting) {
                    return Center(child: CircularProgressIndicator());
                  }
                  
                  if (!snapshot.hasData || snapshot.data!.docs.isEmpty) {
                    return Center(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Icon(Icons.pets, size: 60, color: Colors.grey),
                          SizedBox(height: 20),
                          Text(
                            'لا توجد حيوانات مسجلة بعد',
                            style: TextStyle(fontSize: 18),
                          ),
                          SizedBox(height: 10),
                          Text(
                            'اضغط على زر + لبدء الإضافة',
                            style: TextStyle(fontSize: 14, color: Colors.grey),
                          ),
                        ],
                      ),
                    );
                  }
                  
                  var animals = snapshot.data!.docs;
                  
                  // Apply search filter
                  if (_searchTerm.isNotEmpty) {
                    animals = animals.where((animal) {
                      final data = animal.data() as Map<String, dynamic>;
                      final name = data['الاسم']?.toString().toLowerCase() ?? '';
                      return name.contains(_searchTerm);
                    }).toList();
                  }
                  
                  return ListView.builder(
                    itemCount: animals.length,
                    itemBuilder: (context, index) {
                      final animal = animals[index];
                      final data = animal.data() as Map<String, dynamic>;
                      
                      return AnimalCard(
                        id: animal.id,
                        name: data['الاسم'] ?? 'بدون اسم',
                        type: data['النوع'] ?? 'غير محدد',
                        notes: data['ملاحظات'] ?? 'لا توجد ملاحظات',
                        vaccination: data['التطعيمات'] ?? 'لا توجد تطعيمات',
                        imageUrl: data['رابط_الصورة'] ?? '',
                        date: (data['تاريخ_الإضافة'] as Timestamp).toDate(),
                      );
                    },
                  );
                },
              ),
            ),
          ],
        ),
      ),
    );
  }

  void _showFilterDialog(BuildContext context) {
    String? selectedType;
    
    showDialog(
      context: context,
      builder: (context) {
        return AlertDialog(
          title: Text('تصفية حسب النوع'),
          content: StatefulBuilder(
            builder: (context, setState) {
              return Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  DropdownButton<String>(
                    value: selectedType,
                    hint: Text('اختر نوع الحيوان'),
                    items: ['كل الأنواع', 'بقر', 'جاموس', 'أغنام', 'ماعز', 'إبل', 'خيول']
                        .map((type) => DropdownMenuItem(
                              value: type,
                              child: Text(type),
                            ))
                        .toList(),
                    onChanged: (val) {
                      setState(() => selectedType = val);
                    },
                  ),
                  SizedBox(height: 20),
                  Row(
                    mainAxisAlignment: MainAxisAlignment.end,
                    children: [
                      TextButton(
                        onPressed: () => Navigator.pop(context),
                        child: Text('إلغاء'),
                      ),
                      SizedBox(width: 10),
                      ElevatedButton(
                        onPressed: () {
                          // Apply filter logic
                          Navigator.pop(context);
                        },
                        child: Text('تطبيق'),
                      ),
                    ],
                  ),
                ],
              );
            },
          ),
        );
      },
    );
  }
}

class AnimalCard extends StatelessWidget {
  final String id;
  final String name;
  final String type;
  final String notes;
  final String vaccination;
  final String imageUrl;
  final DateTime date;

  AnimalCard({
    required this.id,
    required this.name,
    required this.type,
    required this.notes,
    required this.vaccination,
    required this.imageUrl,
    required this.date,
  });

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () {
        Navigator.push(
          context,
          MaterialPageRoute(
            builder: (context) => AnimalDetailPage(animalId: id),
          ),
        );
      },
      child: Card(
        elevation: 4,
        margin: EdgeInsets.symmetric(vertical: 8),
        shape: RoundedRectangleBorder(
          borderRadius: BorderRadius.circular(15),
        ),
        child: Padding(
          padding: const EdgeInsets.all(12.0),
          child: Row(
            children: [
              // الصورة
              Hero(
                tag: 'animal-image-$id',
                child: Container(
                  width: 80,
                  height: 80,
                  decoration: BoxDecoration(
                    borderRadius: BorderRadius.circular(10),
                    color: Colors.grey[200],
                  ),
                  child: imageUrl.isNotEmpty
                      ? ClipRRect(
                          borderRadius: BorderRadius.circular(10),
                          child: Image.network(imageUrl, fit: BoxFit.cover),
                        )
                      : Icon(Icons.pets, size: 40, color: Colors.green[700]),
                ),
              ),
              SizedBox(width: 15),
              
              // المعلومات
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        Text(
                          name,
                          style: TextStyle(
                            fontSize: 18,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                        Container(
                          padding: EdgeInsets.symmetric(horizontal: 8, vertical: 4),
                          decoration: BoxDecoration(
                            color: _getTypeColor(type),
                            borderRadius: BorderRadius.circular(10),
                          ),
                          child: Text(
                            type,
                            style: TextStyle(color: Colors.white),
                          ),
                        ),
                      ],
                    ),
                    SizedBox(height: 5),
                    Text(
                      notes,
                      maxLines: 1,
                      overflow: TextOverflow.ellipsis,
                      style: TextStyle(color: Colors.grey[600]),
                    ),
                    SizedBox(height: 5),
                    Text(
                      'تاريخ الإضافة: ${DateFormat('yyyy/MM/dd').format(date)}',
                      style: TextStyle(fontSize: 12, color: Colors.grey),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }

  Color _getTypeColor(String type) {
    switch (type) {
      case 'بقر':
        return Colors.brown;
      case 'جاموس':
        return Colors.blueGrey;
      case 'أغنام':
        return Colors.orange;
      case 'ماعز':
        return Colors.deepOrange;
      case 'إبل':
        return Colors.amber[700]!;
      case 'خيول':
        return Colors.deepPurple;
      default:
        return Colors.green;
    }
  }
}

class AnimalDetailPage extends StatelessWidget {
  final String animalId;

  AnimalDetailPage({required this.animalId});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('تفاصيل الحيوان'),
        actions: [
          IconButton(
            icon: Icon(Icons.edit),
            onPressed: () {
              Navigator.push(
                context,
                MaterialPageRoute(
                  builder: (context) => EditAnimalPage(animalId: animalId),
                ),
              );
            },
          ),
          IconButton(
            icon: Icon(Icons.delete, color: Colors.red),
            onPressed: () {
              _showDeleteConfirmation(context);
            },
          ),
        ],
      ),
      body: FutureBuilder<DocumentSnapshot>(
        future: FirebaseFirestore.instance.collection('الحيوانات').doc(animalId).get(),
        builder: (context, snapshot) {
          if (snapshot.connectionState == ConnectionState.waiting) {
            return Center(child: CircularProgressIndicator());
          }
          
          if (!snapshot.hasData || !snapshot.data!.exists) {
            return Center(child: Text('الحيوان غير موجود'));
          }
          
          final data = snapshot.data!.data() as Map<String, dynamic>;
          
          return SingleChildScrollView(
            padding: EdgeInsets.all(20),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // صورة الحيوان
                Center(
                  child: Hero(
                    tag: 'animal-image-$animalId',
                    child: Container(
                      width: 200,
                      height: 200,
                      decoration: BoxDecoration(
                        borderRadius: BorderRadius.circular(15),
                        boxShadow: [
                          BoxShadow(
                            color: Colors.black26,
                            blurRadius: 10,
                            offset: Offset(0, 5),
                          ),
                        ],
                      ),
                      child: data['رابط_الصورة'] != null && data['رابط_الصورة'].isNotEmpty
                          ? ClipRRect(
                              borderRadius: BorderRadius.circular(15),
                              child: Image.network(
                                data['رابط_الصورة'],
                                fit: BoxFit.cover,
                              ),
                            )
                          : Container(
                              decoration: BoxDecoration(
                                color: Colors.grey[200],
                                borderRadius: BorderRadius.circular(15),
                              ),
                              child: Icon(Icons.pets, size: 60, color: Colors.green),
                            ),
                    ),
                  ),
                ),
                SizedBox(height: 30),
                
                // معلومات الحيوان
                InfoCard(
                  icon: Icons.pets,
                  title: 'معلومات أساسية',
                  children: [
                    InfoRow(label: 'الاسم', value: data['الاسم'] ?? 'غير محدد'),
                    InfoRow(label: 'النوع', value: data['النوع'] ?? 'غير محدد'),
                    InfoRow(
                      label: 'تاريخ الإضافة', 
                      value: DateFormat('yyyy/MM/dd - hh:mm a').format(
                        (data['تاريخ_الإضافة'] as Timestamp).toDate()
                      ),
                    ),
                  ],
                ),
                
                InfoCard(
                  icon: Icons.note,
                  title: 'ملاحظات',
                  children: [
                    Padding(
                      padding: const EdgeInsets.all(12.0),
                      child: Text(
                        data['ملاحظات'] ?? 'لا توجد ملاحظات',
                        style: TextStyle(fontSize: 16),
                    ),
                  ],
                ),
                
                InfoCard(
                  icon: Icons.medical_services,
                  title: 'التطعيمات',
                  children: [
                    Padding(
                      padding: const EdgeInsets.all(12.0),
                      child: Text(
                        data['التطعيمات'] ?? 'لا توجد تطعيمات مسجلة',
                        style: TextStyle(fontSize: 16),
                    ),
                  ],
                ),
              ],
            ),
          );
        },
      ),
    );
  }

  void _showDeleteConfirmation(BuildContext context) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('حذف الحيوان'),
        content: Text('هل أنت متأكد من رغبتك في حذف هذا الحيوان؟ لا يمكن التراجع عن هذه العملية.'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('إلغاء'),
          ),
          TextButton(
            onPressed: () async {
              Navigator.pop(context);
              
              // Delete from Firestore
              await FirebaseFirestore.instance
                  .collection('الحيوانات')
                  .doc(animalId)
                  .delete();
                  
              // Delete image from storage if exists
              final doc = await FirebaseFirestore.instance
                  .collection('الحيوانات')
                  .doc(animalId)
                  .get();
                  
              final data = doc.data() as Map<String, dynamic>;
              if (data['رابط_الصورة'] != null && data['رابط_الصورة'].isNotEmpty) {
                final ref = FirebaseStorage.instance.refFromURL(data['رابط_الصورة']);
                await ref.delete();
              }
              
              Navigator.pop(context);
              ScaffoldMessenger.of(context).showSnackBar(
                SnackBar(
                  content: Text('تم حذف الحيوان بنجاح'),
                  backgroundColor: Colors.green,
                ),
              );
            },
            child: Text('حذف', style: TextStyle(color: Colors.red)),
          ),
        ],
      ),
    );
  }
}

class InfoCard extends StatelessWidget {
  final IconData icon;
  final String title;
  final List<Widget> children;

  InfoCard({required this.icon, required this.title, required this.children});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: EdgeInsets.only(bottom: 20),
      elevation: 3,
      shape: RoundedRectangleBorder(
        borderRadius: BorderRadius.circular(15),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Padding(
            padding: const EdgeInsets.all(16.0),
            child: Row(
              children: [
                Icon(icon, color: Colors.green),
                SizedBox(width: 10),
                Text(
                  title,
                  style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
                ),
              ],
            ),
          ),
          Divider(height: 0),
          ...children,
        ],
      ),
    );
  }
}

class InfoRow extends StatelessWidget {
  final String label;
  final String value;

  InfoRow({required this.label, required this.value});

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      child: Row(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Expanded(
            flex: 2,
            child: Text(
              label,
              style: TextStyle(fontWeight: FontWeight.bold, color: Colors.grey[700]),
            ),
          ),
          Expanded(
            flex: 3,
            child: Text(
              value,
              style: TextStyle(fontSize: 16),
            ),
          ),
        ],
      ),
    );
  }
}

class EditAnimalPage extends StatefulWidget {
  final String animalId;

  EditAnimalPage({required this.animalId});

  @override
  _EditAnimalPageState createState() => _EditAnimalPageState();
}

class _EditAnimalPageState extends State<EditAnimalPage> {
  final GlobalKey<FormState> _formKey = GlobalKey<FormState>();
  late TextEditingController _nameController;
  late TextEditingController _notesController;
  late TextEditingController _vaccinationController;
  
  String? _animalType;
  File? _imageFile;
  String? _currentImageUrl;
  bool _isUploading = false;
  final picker = ImagePicker();

  @override
  void initState() {
    super.initState();
    _loadAnimalData();
  }

  Future<void> _loadAnimalData() async {
    final doc = await FirebaseFirestore.instance
        .collection('الحيوانات')
        .doc(widget.animalId)
        .get();
        
    if (doc.exists) {
      final data = doc.data() as Map<String, dynamic>;
      setState(() {
        _nameController = TextEditingController(text: data['الاسم'] ?? '');
        _notesController = TextEditingController(text: data['ملاحظات'] ?? '');
        _vaccinationController = TextEditingController(text: data['التطعيمات'] ?? '');
        _animalType = data['النوع'];
        _currentImageUrl = data['رابط_الصورة'];
      });
    }
  }

  Future<void> _pickImage() async {
    try {
      final pickedFile = await picker.pickImage(
        source: ImageSource.gallery,
        imageQuality: 85,
        maxWidth: 800,
      );
      if (pickedFile != null) {
        setState(() => _imageFile = File(pickedFile.path));
      }
    } catch (e) {
      _showError('خطأ في اختيار الصورة: ${e.toString()}');
    }
  }

  Future<void> _updateAnimal() async {
    if (!_formKey.currentState!.validate()) return;
    if (_animalType == null) {
      _showError('الرجاء اختيار نوع الحيوان');
      return;
    }

    setState(() => _isUploading = true);

    try {
      String imageUrl = _currentImageUrl ?? '';
      
      // Upload new image if selected
      if (_imageFile != null) {
        // Delete old image if exists
        if (_currentImageUrl != null && _currentImageUrl!.isNotEmpty) {
          final oldRef = FirebaseStorage.instance.refFromURL(_currentImageUrl!);
          await oldRef.delete();
        }
        
        // Upload new image
        final newRef = FirebaseStorage.instance
            .ref()
            .child('صور_الحيوانات')
            .child('${DateTime.now().millisecondsSinceEpoch}.jpg');
        
        await newRef.putFile(_imageFile!);
        imageUrl = await newRef.getDownloadURL();
      }

      // Update Firestore document
      await FirebaseFirestore.instance
          .collection('الحيوانات')
          .doc(widget.animalId)
          .update({
        'الاسم': _nameController.text.trim(),
        'النوع': _animalType,
        'ملاحظات': _notesController.text.trim(),
        'التطعيمات': _vaccinationController.text.trim(),
        'رابط_الصورة': imageUrl,
        'تم_التحديث_في': Timestamp.now(),
      });

      _showSuccess('تم تحديث بيانات الحيوان بنجاح');
      Navigator.pop(context);
    } catch (e) {
      _showError('خطأ في التحديث: ${e.toString()}');
    } finally {
      setState(() => _isUploading = false);
    }
  }

  void _showError(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(message),
        backgroundColor: Colors.red,
      )
    );
  }

  void _showSuccess(String message) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(
        content: Text(message),
        backgroundColor: Colors.green,
      )
    );
  }

  @override
  Widget build(BuildContext context) {
    if (_nameController == null) {
      return Scaffold(
        body: Center(child: CircularProgressIndicator()),
      );
    }
    
    return Scaffold(
      appBar: AppBar(title: Text('تعديل بيانات الحيوان')),
      body: SingleChildScrollView(
        padding: EdgeInsets.all(20),
        child: Form(
          key: _formKey,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              // صورة الحيوان
              Center(
                child: Stack(
                  children: [
                    Container(
                      width: 180,
                      height: 180,
                      decoration: BoxDecoration(
                        color: Colors.grey[200],
                        borderRadius: BorderRadius.circular(15),
                        border: Border.all(color: Colors.green, width: 2),
                      ),
                      child: _imageFile != null
                          ? ClipRRect(
                              borderRadius: BorderRadius.circular(13),
                              child: Image.file(_imageFile!, fit: BoxFit.cover),
                            )
                          : (_currentImageUrl != null && _currentImageUrl!.isNotEmpty
                              ? ClipRRect(
                                  borderRadius: BorderRadius.circular(13),
                                  child: Image.network(_currentImageUrl!, fit: BoxFit.cover),
                                )
                              : Icon(Icons.pets, size: 60, color: Colors.grey)),
                    ),
                    Positioned(
                      bottom: 10,
                      right: 10,
                      child: CircleAvatar(
                        backgroundColor: Colors.green,
                        child: IconButton(
                          icon: Icon(Icons.camera_alt, color: Colors.white),
                          onPressed: _pickImage,
                        ),
                      ),
                    ),
                  ],
                ),
              ),
              SizedBox(height: 30),
              
              // حقل اسم الحيوان
              TextFormField(
                controller: _nameController,
                decoration: InputDecoration(
                  labelText: 'اسم الحيوان أو الرقم',
                  border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.pets),
                ),
                validator: (value) => value!.isEmpty ? 'هذا الحقل مطلوب' : null,
              ),
              SizedBox(height: 20),
              
              // نوع الحيوان
              InputDecorator(
                decoration: InputDecoration(
                  labelText: 'نوع الحيوان',
                  border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.category),
                ),
                child: DropdownButtonHideUnderline(
                  child: DropdownButton<String>(
                    value: _animalType,
                    hint: Text('اختر النوع'),
                    isExpanded: true,
                    items: const [
                      DropdownMenuItem(value: 'بقر', child: Text('بقر')),
                      DropdownMenuItem(value: 'جاموس', child: Text('جاموس')),
                      DropdownMenuItem(value: 'أغنام', child: Text('أغنام')),
                      DropdownMenuItem(value: 'ماعز', child: Text('ماعز')),
                      DropdownMenuItem(value: 'إبل', child: Text('إبل')),
                      DropdownMenuItem(value: 'خيول', child: Text('خيول')),
                    ],
                    onChanged: (val) => setState(() => _animalType = val),
                  ),
                ),
              ),
              SizedBox(height: 20),
              
              // حقل الملاحظات
              TextFormField(
                controller: _notesController,
                decoration: InputDecoration(
                  labelText: 'ملاحظات',
                  border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.note),
                ),
                maxLines: 3,
              ),
              SizedBox(height: 20),
              
              // حقل التطعيمات
              TextFormField(
                controller: _vaccinationController,
                decoration: InputDecoration(
                  labelText: 'التطعيمات',
                  border: OutlineInputBorder(),
                  prefixIcon: Icon(Icons.medical_services),
                ),
                maxLines: 3,
              ),
              SizedBox(height: 30),
              
              // زر الحفظ
              SizedBox(
                width: double.infinity,
                height: 55,
                child: ElevatedButton(
                  onPressed: _isUploading ? null : _updateAnimal,
                  style: ElevatedButton.styleFrom(
                    backgroundColor: Colors.green[700],
                    foregroundColor: Colors.white,
                    shape: RoundedRectangleBorder(
                      borderRadius: BorderRadius.circular(12),
                    ),
                  ),
                  child: _isUploading
                      ? CircularProgressIndicator(color: Colors.white)
                      : Text(
                          'حفظ التعديلات',
                          style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
                        ),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class VaccinationCalendarPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Padding(
        padding: const EdgeInsets.all(20.0),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(
              'تقويم التطعيمات',
              style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 20),
            
            // الشهر الحالي
            Card(
              child: Padding(
                padding: const EdgeInsets.all(16.0),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        Text(
                          'تطعيمات شهر ${DateFormat('MMMM', 'ar').format(DateTime.now())}',
                          style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold),
                        ),
                        Chip(
                          label: Text('12 حيوان'),
                          backgroundColor: Colors.green[100],
                        ),
                      ],
                    ),
                    SizedBox(height: 15),
                    VaccinationList(),
                  ],
                ),
              ),
            ),
            SizedBox(height: 20),
            
            // التطعيمات القادمة
            Text(
              'التطعيمات القادمة',
              style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold),
            ),
            SizedBox(height: 10),
            UpcomingVaccinations(),
          ],
        ),
      ),
    );
  }
}

class VaccinationList extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        VaccinationItem(
          animalName: 'بقرة ١',
          type: 'بقر',
          vaccine: 'تطعيم الحمى القلاعية',
          date: '15 ${DateFormat('MMMM', 'ar').format(DateTime.now())}',
        ),
        VaccinationItem(
          animalName: 'خروف ٣',
          type: 'أغنام',
          vaccine: 'تطعيم الجلد العقدي',
          date: '18 ${DateFormat('MMMM', 'ar').format(DateTime.now())}',
        ),
        VaccinationItem(
          animalName: 'جمل ٢',
          type: 'إبل',
          vaccine: 'تطعيم السعار',
          date: '22 ${DateFormat('MMMM', 'ar').format(DateTime.now())}',
        ),
        VaccinationItem(
          animalName: 'معزة ٥',
          type: 'ماعز',
          vaccine: 'تطعيم الجدري',
          date: '25 ${DateFormat('MMMM', 'ar').format(DateTime.now())}',
        ),
      ],
    );
  }
}

class VaccinationItem extends StatelessWidget {
  final String animalName;
  final String type;
  final String vaccine;
  final String date;

  VaccinationItem({
    required this.animalName,
    required this.type,
    required this.vaccine,
    required this.date,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: EdgeInsets.symmetric(vertical: 8),
      color: Colors.grey[50],
      child: ListTile(
        leading: Container(
          width: 40,
          height: 40,
          decoration: BoxDecoration(
            color: _getTypeColor(type).withOpacity(0.2),
            borderRadius: BorderRadius.circular(10),
          ),
          child: Icon(Icons.medical_services, color: _getTypeColor(type)),
        ),
        title: Text(animalName),
        subtitle: Text(vaccine),
        trailing: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(date, style: TextStyle(fontWeight: FontWeight.bold)),
            Text(type, style: TextStyle(color: Colors.grey)),
          ],
        ),
      ),
    );
  }

  Color _getTypeColor(String type) {
    switch (type) {
      case 'بقر': return Colors.brown;
      case 'أغنام': return Colors.orange;
      case 'إبل': return Colors.amber[700]!;
      case 'ماعز': return Colors.deepOrange;
      default: return Colors.green;
    }
  }
}

class UpcomingVaccinations extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Expanded(
      child: ListView(
        children: [
          UpcomingVaccinationItem(
            animalName: 'بقرة ٣',
            type: 'بقر',
            vaccine: 'تطعيم البروسيلا',
            daysLeft: 5,
          ),
          UpcomingVaccinationItem(
            animalName: 'جاموس ١',
            type: 'جاموس',
            vaccine: 'تطعيم الحمى القلاعية',
            daysLeft: 8,
          ),
          UpcomingVaccinationItem(
            animalName: 'خروف ٧',
            type: 'أغنام',
            vaccine: 'تطعيم الديدان',
            daysLeft: 12,
          ),
          UpcomingVaccinationItem(
            animalName: 'حصان ٢',
            type: 'خيول',
            vaccine: 'تطعيم الإنفلونزا',
            daysLeft: 15,
          ),
        ],
      ),
    );
  }
}

class UpcomingVaccinationItem extends StatelessWidget {
  final String animalName;
  final String type;
  final String vaccine;
  final int daysLeft;

  UpcomingVaccinationItem({
    required this.animalName,
    required this.type,
    required this.vaccine,
    required this.daysLeft,
  });

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: EdgeInsets.only(bottom: 12),
      child: ListTile(
        leading: Container(
          width: 50,
          height: 50,
          decoration: BoxDecoration(
            color: _getTypeColor(type).withOpacity(0.1),
            borderRadius: BorderRadius.circular(10),
          ),
          child: Icon(Icons.medical_services, color: _getTypeColor(type), size: 30),
        ),
        title: Text(animalName),
        subtitle: Text(vaccine),
        trailing: Container(
          padding: EdgeInsets.all(8),
          decoration: BoxDecoration(
            color: _getDaysLeftColor(daysLeft),
            borderRadius: BorderRadius.circular(10),
          ),
          child: Text(
            '$daysLeft أيام',
            style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold),
          ),
        ),
      ),
    );
  }

  Color _getTypeColor(String type) {
    switch (type) {
      case 'بقر': return Colors.brown;
      case 'جاموس': return Colors.blueGrey;
      case 'أغنام': return Colors.orange;
      case 'خيول': return Colors.deepPurple;
      default: return Colors.green;
    }
  }

  Color _getDaysLeftColor(int days) {
    if (days <= 3) return Colors.red;
    if (days <= 7) return Colors.orange;
    return Colors.green;
  }
}

class AnimalSearchDelegate extends SearchDelegate {
  @override
  List<Widget> buildActions(BuildContext context) {
    return [
      IconButton(
        icon: Icon(Icons.clear),
        onPressed: () {
          query = '';
        },
      ),
    ];
  }

  @override
  Widget buildLeading(BuildContext context) {
    return IconButton(
      icon: Icon(Icons.arrow_back),
      onPressed: () {
        close(context, null);
      },
    );
  }

  @override
  Widget buildResults(BuildContext context) {
    return _buildSearchResults();
  }

  @override
  Widget buildSuggestions(BuildContext context) {
    return _buildSearchResults();
  }

  Widget _buildSearchResults() {
    return StreamBuilder<QuerySnapshot>(
      stream: FirebaseFirestore.instance.collection('الحيوانات').snapshots(),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return Center(child: CircularProgressIndicator());
        }

        final results = snapshot.data!.docs.where((doc) {
          final data = doc.data() as Map<String, dynamic>;
          final name = data['الاسم']?.toString().toLowerCase() ?? '';
          return name.contains(query.toLowerCase());
        }).toList();

        if (results.isEmpty) {
          return Center(
            child: Text(
              'لا توجد نتائج لـ "$query"',
              style: TextStyle(fontSize: 18),
            ),
          );
        }

        return ListView.builder(
          itemCount: results.length,
          itemBuilder: (context, index) {
            final animal = results[index];
            final data = animal.data() as Map<String, dynamic>;
            
            return ListTile(
              leading: data['رابط_الصورة'] != null && data['رابط_الصورة'].isNotEmpty
                  ? CircleAvatar(
                      backgroundImage: NetworkImage(data['رابط_الصورة']),
                    )
                  : CircleAvatar(
                      child: Icon(Icons.pets),
                    ),
              title: Text(data['الاسم'] ?? 'بدون اسم'),
              subtitle: Text(data['النوع'] ?? 'غير محدد'),
              onTap: () {
                Navigator.push(
                  context,
                  MaterialPageRoute(
                    builder: (context) => AnimalDetailPage(animalId: animal.id),
                  ),
                );
              },
            );
          },
        );
      },
    );
  }
}
