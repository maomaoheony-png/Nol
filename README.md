// main.dart // تطبيق Flutter MVP بسيط لتواصل اجتماعي اسلامي مع "خاتم" فحص المحتوى // ملاحظات: // - هذا نموذج أولي (prototype) للتجربة المحلية. ليس للاستخدام الإنتاجي دون إضافة مصادقة حقيقية، تخزين سحابي، ونماذج فحص أقوى. // - القواعد الشرعية / قائمة الكلمات المحظورة قابلة للتعديل في القاموس bannedWords. // - يوجد واجهة "مشرف" بسيطة للدخول بكلمة مرور ثابتة (moderator) // - اللغة في الواجهة عربية. عدل النصوص حسب لهجتك.

import 'package:flutter/material.dart';

void main() { runApp(IslamicSocialApp()); }

class IslamicSocialApp extends StatelessWidget { @override Widget build(BuildContext context) { return MaterialApp( title: 'تواصل اسلامي - خاتم', theme: ThemeData( primarySwatch: Colors.green, fontFamily: 'Helvetica', ), home: LoginPage(), debugShowCheckedModeBanner: false, ); } }

// ---------------------------- // بيانات مبدئية في الذاكرة (MVP) // ----------------------------

class Post { String id; String author; String text; DateTime createdAt; String status; // 'approved', 'rejected', 'pending' String rejectReason;

Post({required this.id, required this.author, required this.text, DateTime? createdAt, this.status = 'pending', this.rejectReason = ''}) : this.createdAt = createdAt ?? DateTime.now(); }

// قائمة كلمات محظورة مبدئية. المستخدم/الادارة تقدر تعدلها List<String> bannedWords = [ // أمثلة -- عدل حسب القواعد الشرعية اللي تريدها (فتاوى السيد السيستاني لو تحب) 'فاحشة', 'خلاعة', 'عري', 'سب', 'شتيمة', ];

// قاعدة بيانات مبدئية بالذاكرة class InMemoryDB { List<Post> posts = []; List<Post> moderationQueue = []; // منشورات تحتاج مراجعة بشرية }

final db = InMemoryDB();

// ---------------------------- // صفحة الدخول (باسم بسيط دون باك اند) // ----------------------------

class LoginPage extends StatefulWidget { @override _LoginPageState createState() => _LoginPageState(); }

class _LoginPageState extends State<LoginPage> { final TextEditingController _nameCtrl = TextEditingController();

void _login() { String name = nameCtrl.text.trim(); if (name.isEmpty) { ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('اكتب اسم المستخدم'))); return; } Navigator.pushReplacement( context, MaterialPageRoute(builder: () => HomePage(username: name)), ); }

@override Widget build(BuildContext context) { return Scaffold( body: SafeArea( child: Padding( padding: const EdgeInsets.all(20.0), child: Column( mainAxisAlignment: MainAxisAlignment.center, children: [ Text('تطبيق تواصل اسلامي', style: TextStyle(fontSize: 28, fontWeight: FontWeight.bold)), SizedBox(height: 10), Text('خاتم يقوم بفحص المحتوى قبل النشر', style: TextStyle(fontSize: 16)), SizedBox(height: 30), TextField( controller: _nameCtrl, decoration: InputDecoration(border: OutlineInputBorder(), labelText: 'اسم المستخدم'), ), SizedBox(height: 12), ElevatedButton(onPressed: login, child: Text('دخول')), SizedBox(height: 20), TextButton( onPressed: () { // اختصار لدخول المشرف Navigator.push(context, MaterialPageRoute(builder: () => ModeratorLoginPage())); }, child: Text('دخول كمشرف'), ) ], ), ), ), ); } }

// ---------------------------- // الصفحة الرئيسية - الخلاصة // ----------------------------

class HomePage extends StatefulWidget { final String username; HomePage({required this.username});

@override _HomePageState createState() => _HomePageState(); }

class _HomePageState extends State<HomePage> { @override void initState() { super.initState(); // في MVP، بعض المنشورات التجريبية if (db.posts.isEmpty) { db.posts.addAll([ Post(id: 'p1', author: 'ادم', text: 'مرحبا بكم في التطبيق', status: 'approved'), Post(id: 'p2', author: 'سارة', text: 'احنا نحب الخير والمساعدة', status: 'approved'), ]); } }

void openNewPost() async { final result = await Navigator.push(context, MaterialPageRoute(builder: () => NewPostPage(author: widget.username))); if (result == true) setState(() {}); }

void openModerationQueue() { Navigator.push(context, MaterialPageRoute(builder: () => ModeratorPanelPage())); }

@override Widget build(BuildContext context) { List<Post> visible = db.posts.where((p) => p.status == 'approved' || p.author == widget.username).toList();

return Scaffold(
  appBar: AppBar(
    title: Text('الخلاصة - مرحبا ${widget.username}'),
    actions: [
      IconButton(onPressed: _openModerationQueue, icon: Icon(Icons.admin_panel_settings)),
    ],
  ),
  body: ListView.builder(
    itemCount: visible.length,
    itemBuilder: (c, i) {
      final post = visible[i];
      return Card(
        margin: EdgeInsets.all(8),
        child: ListTile(
          title: Text(post.author + ' — ' + post.text),
          subtitle: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(post.createdAt.toLocal().toString()),
              if (post.status != 'approved') Text('حالة: ${post.status} ${post.rejectReason.isNotEmpty ? '- سبب: ${post.rejectReason}' : ''}', style: TextStyle(color: Colors.red)),
            ],
          ),
        ),
      );
    },
  ),
  floatingActionButton: FloatingActionButton(
    onPressed: _openNewPost,
    child: Icon(Icons.add),
    tooltip: 'منشور جديد',
  ),
);

} }

// ---------------------------- // صفحة كتابة منشور جديد // ----------------------------

class NewPostPage extends StatefulWidget { final String author; NewPostPage({required this.author});

@override _NewPostPageState createState() => _NewPostPageState(); }

class _NewPostPageState extends State<NewPostPage> { final TextEditingController _textCtrl = TextEditingController();

void _submit() { String text = _textCtrl.text.trim(); if (text.isEmpty) { ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('اكتب نص المنشور'))); return; }

// الخاتم: فحص بسيط للنص
final result = simpleTextModeration(text);

final post = Post(id: DateTime.now().millisecondsSinceEpoch.toString(), author: widget.author, text: text);

if (result['decision'] == 'reject') {
  post.status = 'rejected';
  post.rejectReason = result['reason'] as String;
  db.posts.add(post);
  ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('تم رفض المنشور: ${post.rejectReason}')));
  Navigator.pop(context, true);
  return;
}

if (result['decision'] == 'approve') {
  post.status = 'approved';
  db.posts.add(post);
  ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('تم نشر المنشور')));
  Navigator.pop(context, true);
  return;
}

// مشتبه -> أرسل للمراجعة البشرية
post.status = 'pending';
db.moderationQueue.add(post);
db.posts.add(post);
ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('المنشور مرسل للمراجعة البشرية')));
Navigator.pop(context, true);

}

@override Widget build(BuildContext context) { return Scaffold( appBar: AppBar(title: Text('منشور جديد')), body: Padding( padding: const EdgeInsets.all(16.0), child: Column( children: [ TextField( controller: _textCtrl, maxLines: 6, decoration: InputDecoration(border: OutlineInputBorder(), hintText: 'اكتب نصك هنا...'), ), SizedBox(height: 12), ElevatedButton(onPressed: _submit, child: Text('انشر')), ], ), ), ); } }

// ---------------------------- // محرك فحص نصي بسيط (الخاتم MVP) // - يعيد خريطة تحتوي decision: approve/reject/pending // - reason: سبب الرفض او تعليق // ----------------------------

Map<String, dynamic> simpleTextModeration(String text) { final lower = text.toLowerCase();

// فحص مباشر لكلمات ممنوعة for (var w in bannedWords) { if (lower.contains(w)) { return {'decision': 'reject', 'reason': 'احتوى على كلمة محظورة: $w'}; } }

// قواعد بسيطة لاكتشاف التلميح الجنسي — مثال (يمكن تحسينه بنماذج ML) final sexualHints = ['رغبة', 'مثير', 'علاقة محرمة']; int hintCount = 0; for (var s in sexualHints) if (lower.contains(s)) hintCount++; if (hintCount >= 1) { // نضعها كمشتبه return {'decision': 'pending', 'reason': 'مشتبه: قد يحتوي تلميح جنسي'}; }

// افتراض افتراضي: موافق return {'decision': 'approve', 'reason': ''}; }

// ---------------------------- // صفحات المشرفين (moderation) // ----------------------------

class ModeratorLoginPage extends StatefulWidget { @override _ModeratorLoginPageState createState() => _ModeratorLoginPageState(); }

class _ModeratorLoginPageState extends State<ModeratorLoginPage> { final TextEditingController _pwd = TextEditingController(); final String modPassword = 'mod1234'; // غيّرها لاحقاً

void _tryLogin() { if (pwd.text == modPassword) { Navigator.pushReplacement(context, MaterialPageRoute(builder: () => ModeratorPanelPage())); } else { ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('كلمة المرور خطأ'))); } }

@override Widget build(BuildContext context) { return Scaffold( appBar: AppBar(title: Text('دخول المشرف')), body: Padding( padding: EdgeInsets.all(16), child: Column( children: [ TextField(controller: _pwd, obscureText: true, decoration: InputDecoration(labelText: 'كلمة المرور')), SizedBox(height: 12), ElevatedButton(onPressed: _tryLogin, child: Text('دخول')), ], ), ), ); } }

class ModeratorPanelPage extends StatefulWidget { @override _ModeratorPanelPageState createState() => _ModeratorPanelPageState(); }

class _ModeratorPanelPageState extends State<ModeratorPanelPage> { void _approve(Post p) { setState(() { p.status = 'approved'; db.moderationQueue.removeWhere((x) => x.id == p.id); }); }

void _reject(Post p, String reason) { setState(() { p.status = 'rejected'; p.rejectReason = reason; db.moderationQueue.removeWhere((x) => x.id == p.id); }); }

@override Widget build(BuildContext context) { return Scaffold( appBar: AppBar(title: Text('لوحة المراقبة')), body: Padding( padding: EdgeInsets.all(12), child: Column( children: [ Text('منشورات بانتظار المراجعة', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)), Expanded( child: ListView.builder( itemCount: db.posts.length, itemBuilder: (c, i) { final post = db.posts[i]; if (post.status != 'pending') return SizedBox.shrink(); return Card( margin: EdgeInsets.symmetric(vertical: 8), child: ListTile( title: Text(post.author + ': ' + post.text), subtitle: Text(post.createdAt.toLocal().toString()), trailing: Row( mainAxisSize: MainAxisSize.min, children: [ IconButton( icon: Icon(Icons.check, color: Colors.green), onPressed: () => _approve(post), ), IconButton( icon: Icon(Icons.close, color: Colors.red), onPressed: () async { final reason = await _askRejectReason(context); if (reason != null) _reject(post, reason); }, ), ], ), ), ); }, ), ), SizedBox(height: 10), Text('قائمة الكلمات المحظورة (تعديل مباشر للمصدار)', style: TextStyle(fontWeight: FontWeight.bold)), Wrap( spacing: 8, children: bannedWords.map((w) => Chip(label: Text(w))).toList(), ), ], ), ), ); }

Future<String?> askRejectReason(BuildContext ctx) { final TextEditingController ctrl = TextEditingController(); return showDialog<String>( context: ctx, builder: () => AlertDialog( title: Text('سبب الرفض'), content: TextField(controller: ctrl, decoration: InputDecoration(hintText: 'اكتب سبب الرفض')), actions: [ TextButton(onPressed: () => Navigator.pop(ctx, null), child: Text('الغاء')), ElevatedButton(onPressed: () => Navigator.pop(ctx, ctrl.text.trim()), child: Text('رفض')), ], ), ); } }

// نهاية الملف

