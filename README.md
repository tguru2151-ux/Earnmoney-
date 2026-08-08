name: earn_money
description: "Earn Money App - No Ads, No Rewarded Video"
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.3.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.6.0
  firebase_auth: ^5.3.1
  cloud_firestore: ^5.4.3
  google_sign_in: ^6.2.1
  intl: ^0.19.0
  url_launcher: ^6.3.0

flutter:
  uses-material-design: true
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:intl/intl.dart';
import 'package:url_launcher/url_launcher.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(EarnMoneyApp());
}

const double COIN_RATE = 100; // 100 Coins = ₹1
const String ADMIN_PASSWORD = "@Nikhil001";

class EarnMoneyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Earn Money',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(primarySwatch: Colors.green, scaffoldBackgroundColor: Colors.white, appBarTheme: AppBarTheme(backgroundColor: Colors.green, foregroundColor: Colors.white), elevatedButtonTheme: ElevatedButtonThemeData(style: ElevatedButton.styleFrom(backgroundColor: Colors.green, foregroundColor: Colors.white, minimumSize: Size(double.infinity, 50))), useMaterial3: true),
      home: AuthWrapper(),
    );
  }
}

class AuthWrapper extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return StreamBuilder<User?>(
      stream: FirebaseAuth.instance.authStateChanges(),
      builder: (context, snapshot) {
        if (snapshot.connectionState == ConnectionState.waiting) return Scaffold(body: Center(child: CircularProgressIndicator()));
        if (snapshot.hasData) return HomePage();
        return LoginPage();
      },
    );
  }
}

class LoginPage extends StatefulWidget {
  @override
  State<LoginPage> createState() => _LoginPageState();
}
class _LoginPageState extends State<LoginPage> {
  final emailController = TextEditingController(); final passController = TextEditingController(); bool loading = false;
  Future<void> createUserIfNotExists() async {
    final user = FirebaseAuth.instance.currentUser!; final doc = await FirebaseFirestore.instance.collection('USERS').doc(user.uid).get();
    if (!doc.exists) { String referCode = user.uid.substring(0, 6).toUpperCase(); await FirebaseFirestore.instance.collection('USERS').doc(user.uid).set({'Email': user.email, 'Name': user.displayName ?? 'User', 'Photo': user.photoURL ?? '', 'Coins': 0, 'BalanceINR': 0.0, 'ReferCode': referCode, 'ReferredBy': '', 'TotalEarned': 0.0, 'TotalWithdrawn': 0.0, 'ReferCount': 0, 'ReferComplete': 0, 'WithdrawToday': 0, 'LastDailyClaim': '', }); }
  }
  Future<void> signInWithGoogle() async { setState(()=>loading=true); final googleUser = await GoogleSignIn().signIn(); if (googleUser == null) { setState(()=>loading=false); return; } final googleAuth = await googleUser.authentication; final credential = GoogleAuthProvider.credential(accessToken: googleAuth.accessToken, idToken: googleAuth.idToken); await FirebaseAuth.instance.signInWithCredential(credential); await createUserIfNotExists(); setState(()=>loading=false); }
  Future<void> signInWithEmail() async { try { setState(()=>loading=true); await FirebaseAuth.instance.signInWithEmailAndPassword(email: emailController.text.trim(), password: passController.text.trim()); await createUserIfNotExists(); } catch(e) { try { await FirebaseAuth.instance.createUserWithEmailAndPassword(email: emailController.text.trim(), password: passController.text.trim()); await createUserIfNotExists(); } catch(e2){ ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(e2.toString()))); } } setState(()=>loading=false); }
  @override
  Widget build(BuildContext context) { return Scaffold(body: Center(child: SingleChildScrollView(padding: EdgeInsets.all(20), child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [ Text("Earn Money", style: TextStyle(fontSize: 32, fontWeight: FontWeight.bold, color: Colors.green)), SizedBox(height: 30), TextField(controller: emailController, decoration: InputDecoration(labelText: "Email", border: OutlineInputBorder())), SizedBox(height: 10), TextField(controller: passController, decoration: InputDecoration(labelText: "Password", border: OutlineInputBorder()), obscureText: true), SizedBox(height: 20), if(loading) CircularProgressIndicator() else Column(children:[ ElevatedButton(onPressed: signInWithEmail, child: Text("Login / Sign Up")), SizedBox(height: 10), ElevatedButton.icon(icon: Icon(Icons.g_mobiledata), label: Text("Sign In with Google"), onPressed: signInWithGoogle), TextButton(onPressed: ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => AdminLogin())), child: Text("Admin Login", style: TextStyle(color: Colors.red))) ])])))); }
}

class HomePage extends StatelessWidget {
  Future<void> claimDaily(BuildContext context, String uid, Map user) async {
    String today = DateFormat('yyyy-MM-dd').format(DateTime.now()); if (user['LastDailyClaim'] == today) { ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text("Already claimed today"))); return; }
    double newCoins = (user['Coins'] + 10).toDouble(); double newBalance = newCoins / COIN_RATE; await FirebaseFirestore.instance.collection('USERS').doc(uid).update({'Coins': newCoins, 'BalanceINR': newBalance, 'LastDailyClaim': today, 'TotalEarned': (user['TotalEarned'] + 0.10).toDouble()}); await FirebaseFirestore.instance.collection('HISTORY').add({'Email': user['Email'], 'Type': 'Daily Bonus', 'Amount': 10, 'Date': Timestamp.now(), 'Status': 'Complete'});
  }
  @override
  Widget build(BuildContext context) {
    final user = FirebaseAuth.instance.currentUser!; return StreamBuilder<DocumentSnapshot>(stream: FirebaseFirestore.instance.collection('USERS').doc(user.uid).snapshots(), builder: (context, snapshot) {
      if (!snapshot.hasData) return Scaffold(body: Center(child: CircularProgressIndicator())); var data = snapshot.data!.data() as Map; return Scaffold(appBar: AppBar(title: Text("Welcome ${data['Name']}")), body: Padding(padding: EdgeInsets.all(16), child: Column(children: [ Card(color: Colors.green.shade50, child: Padding(padding: EdgeInsets.all(16), child: Column(children:[ Text("Current Balance: ₹${data['BalanceINR'].toStringAsFixed(2)}", style: TextStyle(fontSize: 24, fontWeight: FontWeight.bold)), Text("Total Coins: ${data['Coins']}"), ]))), SizedBox(height: 20), Expanded(child: GridView.count(crossAxisCount: 2, crossAxisSpacing: 10, mainAxisSpacing: 10, children: [ _buildButton(context, "Claim Daily 10 Coins", Icons.card_giftcard, ()=>claimDaily(context, user.uid, data)), _buildButton(context, "Offers", Icons.local_offer, ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => TaskPage("OFFERS")))), _buildButton(context, "Surveys", Icons.poll, ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => TaskPage("SURVEYS")))), _buildButton(context, "Wallet", Icons.account_balance_wallet, ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => WalletPage(data)))), _buildButton(context, "Refer", Icons.people, ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => ReferPage(data)))), _buildButton(context, "History", Icons.history, ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => HistoryPage()))), _buildButton(context, "Profile", Icons.person, ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => ProfilePage(data)))), ])) ])), ); }); }
  Widget _buildButton(context, title, icon, onTap) { return Card(elevation: 2, child: InkWell(onTap: onTap, child: Column(mainAxisAlignment: MainAxisAlignment.center, children:[Icon(icon, size: 40, color: Colors.green), SizedBox(height: 5), Text(title, textAlign: TextAlign.center)]))); }
}

class TaskPage extends StatelessWidget {
  final String collection; TaskPage(this.collection);
  Future<void> completeTask(Map task, Map user) async { final uid = FirebaseAuth.instance.currentUser!.uid; final userRef = FirebaseFirestore.instance.collection('USERS').doc(uid); double coins = task['Coins'].toDouble(); double newCoins = user['Coins'] + coins; await userRef.update({'Coins': newCoins, 'BalanceINR': newCoins / COIN_RATE, 'TotalEarned': user['TotalEarned'] + (coins / COIN_RATE)}); await FirebaseFirestore.instance.collection('HISTORY').add({'Email': user['Email'], 'Type': collection, 'Amount': coins, 'Date': Timestamp.now(), 'Status': 'Complete'}); if(await canLaunchUrl(Uri.parse(task['Link']))) launchUrl(Uri.parse(task['Link'])); }
  @override
  Widget build(BuildContext context) {
    final user = FirebaseAuth.instance.currentUser!; return StreamBuilder<DocumentSnapshot>(stream: FirebaseFirestore.instance.collection('USERS').doc(user.uid).snapshots(), builder: (context, userSnap) {
      if(!userSnap.hasData) return Scaffold(body: Center(child: CircularProgressIndicator())); var userData = userSnap.data!.data() as Map; return StreamBuilder<QuerySnapshot>(stream: FirebaseFirestore.instance.collection(collection).snapshots(), builder: (context, snapshot) { return Scaffold(appBar: AppBar(title: Text(collection)), body: snapshot.hasData ? ListView(children: snapshot.data!.docs.map((doc) { var d = doc.data() as Map; return Card(child: ListTile(title: Text(d['Title']), subtitle: Text("${d['Coins']} Coins - ${d['Description']}"), trailing: ElevatedButton(onPressed: ()=>completeTask(d, userData), child: Text("Earn")))); }).toList()) : Center(child: CircularProgressIndicator()), ); }); }
}

class WalletPage extends StatelessWidget { final Map data; WalletPage(this.data); @override Widget build(BuildContext context) { return Scaffold(appBar: AppBar(title: Text("Wallet")), body: Padding(padding: EdgeInsets.all(16), child: Column(children: [ Text("Total Earned: ₹${data['TotalEarned'].toStringAsFixed(2)}", style: TextStyle(fontSize: 18)), Text("Total Withdrawn: ₹${data['TotalWithdrawn'].toStringAsFixed(2)}", style: TextStyle(fontSize: 18)), Text("Rate: 100 Coins = ₹1", style: TextStyle(fontSize: 16)), SizedBox(height: 20), ElevatedButton(onPressed: ()=>Navigator.push(context, MaterialPageRoute(builder: (_) => WithdrawPage(data))), child: Text("Withdraw Money")) ])), ); } }

class WithdrawPage extends StatefulWidget { final Map data; WithdrawPage(this.data); @override State<WithdrawPage> createState() => _WithdrawPageState(); }
class _WithdrawPageState extends State<WithdrawPage> { String type = "UPI"; int amount = 2; final upiController = TextEditingController();
  Future<void> requestWithdraw() async { final uid = FirebaseAuth.instance.currentUser!.uid; final userRef = FirebaseFirestore.instance.collection('USERS').doc(uid); int deductCoins = amount * 100; await userRef.update({'Coins': widget.data['Coins'] - deductCoins, 'BalanceINR': (widget.data['Coins'] - deductCoins) / COIN_RATE, 'WithdrawToday': widget.data['WithdrawToday'] + 1, }); await FirebaseFirestore.instance.collection('WITHDRAWALS').add({'Email': widget.data['Email'], 'Type': type, 'AmountINR': amount, 'UPIorEmail': upiController.text, 'Date': Timestamp.now(), 'Status': 'Pending'}); await FirebaseFirestore.instance.collection('HISTORY').add({'Email': widget.data['Email'], 'Type': 'Withdrawal', 'Amount': -deductCoins, 'Date': Timestamp.now(), 'Status': 'Pending'}); ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text("Withdrawal Requested: ₹$amount"))); Navigator.pop(context); }
  @override Widget build(BuildContext context) { bool canWithdraw = widget.data['WithdrawToday'] < 2 && widget.data['BalanceINR'] >= amount; return Scaffold(appBar: AppBar(title: Text("Withdrawal")), body: Padding(padding: EdgeInsets.all(16), child: Column(children: [ Text("Note: Min 2 withdrawals required to activate Refer Bonus"), Text("Per Day Limit: 2 Withdrawals. Used: ${widget.data['WithdrawToday']}"), SizedBox(height: 10), DropdownButtonFormField(value: type, decoration: InputDecoration(labelText: "Withdrawal Type", border: OutlineInputBorder()), items: ["UPI","Amazon Gift Card","Play Store Gift Card"].map((e)=>DropdownMenuItem(value:e,child:Text(e))).toList(), onChanged: (v)=>setState(()=>type=v!)), SizedBox(height: 10), DropdownButtonFormField(value: amount, decoration: InputDecoration(labelText: "Amount", border: OutlineInputBorder()), items: [2,3,5,8,10,20].map((e)=>DropdownMenuItem(value:e,child:Text("₹$e"))).toList(), onChanged: (v)=>setState(()=>amount=v!)), SizedBox(height: 10), TextField(controller: upiController, decoration: InputDecoration(labelText: "UPI ID / Gift Card Email", border: OutlineInputBorder())), SizedBox(height: 20), if(canWithdraw) ElevatedButton(onPressed: requestWithdraw, child: Text("Request Withdraw")) else Text("Not eligible", style: TextStyle(color: Colors.red)) ])), ); } }

class ReferPage extends StatelessWidget { final Map data; ReferPage(this.data); @override Widget build(BuildContext context) { return Scaffold(appBar: AppBar(title: Text("Refer & Earn")), body: Padding(padding: EdgeInsets.all(16), child: Column(crossAxisAlignment: CrossAxisAlignment.start, children: [ Text("Your Code: ${data['ReferCode']}", style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)), Text("Total Invited: ${data['ReferCount']}"), Text("Completed: ${data['ReferComplete']}"), Text("Earned from Refer: ₹${(data['TotalEarned'] - data['TotalWithdrawn'] - data['Coins']/COIN_RATE).toStringAsFixed(2)}"), Text("Rule: Get 250 Coins = ₹2.50 when your friend does 2 withdrawals"), Divider(), Text("Referred Users:", style: TextStyle(fontWeight: FontWeight.bold)), Expanded(child: StreamBuilder<QuerySnapshot>(stream: FirebaseFirestore.instance.collection('USERS').where('ReferredBy', isEqualTo: data['ReferCode']).snapshots(), builder: (context, snap) => snap.hasData ? ListView(children: snap.data!.docs.map((d)=>ListTile(title:Text(d['Name']))).toList()) : CircularProgressIndicator())) ])), ); } }

class HistoryPage extends StatelessWidget { @override Widget build(BuildContext context) { final email = FirebaseAuth.instance.currentUser!.email; return Scaffold(appBar: AppBar(title: Text("History")), body: StreamBuilder<QuerySnapshot>(stream: FirebaseFirestore.instance.collection('HISTORY').where('Email', isEqualTo: email).orderBy('Date', descending: true).snapshots(), builder: (context, snap) => snap.hasData ? ListView(children: snap.data!.docs.map((d){ var data = d.data() as Map; return ListTile(title:Text("${data['Type']} - ${data['Amount']}"), subtitle: Text("${data['Status']} - ${DateFormat('dd MMM yy').format(data['Date'].toDate())}")); }).toList()) : Center(child: CircularProgressIndicator()))); } }
class ProfilePage extends StatelessWidget { final Map data; ProfilePage(this.data); @override Widget build(BuildContext context) { return Scaffold(appBar: AppBar(title: Text("Profile")), body: Center(child: Column(children: [ SizedBox(height: 20), CircleAvatar(backgroundImage: NetworkImage(data['Photo'].isEmpty ? 'https://i.pravatar.cc/150' : data['Photo']), radius: 40), SizedBox(height: 10), Text(data['Name'], style: TextStyle(fontSize: 20)), Text(data['Email']), SwitchListTile(title: Text("Notification"), value: true, onChanged: (_){}), Spacer(), ElevatedButton(onPressed: ()=>FirebaseAuth.instance.signOut(), child: Text("Logout")), SizedBox(height: 20) ])), ); } }

class AdminLogin extends StatefulWidget { @override State<AdminLogin> createState() => _AdminLoginState(); }
class _AdminLoginState extends State<AdminLogin> { final pass = TextEditingController(); void login() { if(pass.text == ADMIN_PASSWORD) { Navigator.push(context, MaterialPageRoute(builder: (_) => AdminPanel())); } else { ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text("Wrong Password"))); } @override Widget build(BuildContext context) { return Scaffold(appBar: AppBar(title: Text("Admin Login")), body: Padding(padding: EdgeInsets.all(20), child: Column(mainAxisAlignment: MainAxisAlignment.center, children: [ TextField(controller: pass, decoration: InputDecoration(labelText: "Admin Password"), obscureText: true), SizedBox(height: 20), ElevatedButton(onPressed: login, child: Text("Login")) ]))); } }

class AdminPanel extends StatelessWidget {
  Future<void> updateWithdraw(String id, String status, Map w) async {
    await FirebaseFirestore.instance.collection('WITHDRAWALS').doc(id).update({'Status': status});
    if(status == "Rejected") { final userQuery = await FirebaseFirestore.instance.collection('USERS').where('Email', isEqualTo: w['Email']).get(); if(userQuery.docs.isNotEmpty) { final user = userQuery.docs.first.data(); int addCoins = w['AmountINR'] * 100; await userQuery.docs.first.reference.update({'Coins': user['Coins'] + addCoins, 'BalanceINR': (user['Coins'] + addCoins) / COIN_RATE}); } }
    if(status == "Completed") { await FirebaseFirestore.instance.collection('USERS').where('Email', isEqualTo: w['Email']).get().then((snap){ var user = snap.docs.first.data(); snap.docs.first.reference.update({'TotalWithdrawn': user['TotalWithdrawn'] + w['AmountINR']}); }); await FirebaseFirestore.instance.collection('HISTORY').where('Email', isEqualTo: w['Email']).where('Status', isEqualTo: 'Pending').limit(1).get().then((snap){ if(snap.docs.isNotEmpty) snap.docs.first.reference.update({'Status':'Complete'}); }); }
  }
  @override
  Widget build(BuildContext context) {
    return Scaffold(appBar: AppBar(title: Text("Admin Panel")), body: StreamBuilder<QuerySnapshot>(stream: FirebaseFirestore.instance.collection('WITHDRAWALS').orderBy('Date', descending: true).snapshots(), builder: (context, snap) => snap.hasData ? ListView(children: snap.data!.docs.map((d){ var w = d.data() as Map; return Card(child: ListTile(title: Text("${w['Email']} - ₹${w['AmountINR']}"), subtitle: Text("${w['Type']} - ${w['Status']}\n${w['UPIorEmail']}"), trailing: w['Status'] == 'Pending' ? Row(mainAxisSize: MainAxisSize.min, children: [ IconButton(icon: Icon(Icons.check, color: Colors.green), onPressed: ()=>updateWithdraw(d.id, "Completed", w)), IconButton(icon: Icon(Icons.close, color: Colors.red), onPressed: ()=>updateWithdraw(d.id, "Rejected", w)), ]) : Chip(label: Text(w['Status']))), )); }).toList()) : Center(child: CircularProgressIndicator()))); }
}
# Earn Money App

Flutter + Firebase Earning App
No Ads | No Rewarded Video

## Features
- Google + Email/Password Login
- Daily 10 Coins Claim
- Offers & Surveys System
- Wallet + Withdrawal UPI/Gift Card: ₹2,3,5,8,10,20
- Refer & Earn System  
- Admin Panel with Password: @Nikhil001
- Theme: Green + White

## Firebase Setup
1. Create project in Firebase
2. Enable Authentication: Google + Email/Password
3. Create Firestore Database
4. Create 5 Collections: `USERS, HISTORY, OFFERS, SURVEYS, WITHDRAWALS`
5. Add `google-services.json` in android/app

## Rate
100 Coins = ₹1
git init
git add .
git commit -m "Earn Money App Full Code with UPI 2,3,5,8,10,20"
git branch -M main
git remote add origin https://github.com/tumhara-username/earn-money-app.git
git push -u origin main
