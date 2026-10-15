name: Build APK
on:
  workflow_dispatch:
  push:
    branches: [main, master]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
      - uses: subosito/flutter-action@v2
        with:
          channel: stable
      - name: Build
        run: |
          flutter create --project-name geoquiz --org com.mapx --platforms android .
          rm -rf test
          flutter pub add google_mobile_ads
          # معرّف تطبيق AdMob التجريبي - استبدله بمعرّفك الحقيقي لاحقاً
          sed -i 's|</application>|<meta-data android:name="com.google.android.gms.ads.APPLICATION_ID" android:value="ca-app-pub-7350231870045123~8577724868"/></application>|' android/app/src/main/AndroidManifest.xml
          sed -i 's/minSdk = flutter.minSdkVersion/minSdk = 24/; s/minSdkVersion flutter.minSdkVersion/minSdkVersion 24/' android/app/build.gradle*
          cat > lib/main.dart << 'EOF'
          import 'package:flutter/material.dart';
          import 'package:google_mobile_ads/google_mobile_ads.dart';

          // معرّف وحدة البانر التجريبي - استبدله بمعرّفك الحقيقي لاحقاً
          const bannerId = 'ca-app-pub-7350231870045123/4638479853';

          void main() {
            WidgetsFlutterBinding.ensureInitialized();
            MobileAds.instance.initialize();
            runApp(const GeoApp());
          }

          // السؤال ثم الإجابة الصحيحة أولاً ثم الخيارات الخاطئة
          const data = [
            ['ما عاصمة المغرب؟', 'الرباط', 'الدار البيضاء', 'مراكش', 'فاس'],
            ['ما أكبر قارة في العالم؟', 'آسيا', 'أفريقيا', 'أوروبا', 'أمريكا الشمالية'],
            ['ما أطول نهر في أفريقيا؟', 'النيل', 'الكونغو', 'النيجر', 'زامبيزي'],
            ['ما أكبر دولة في العالم مساحة؟', 'روسيا', 'كندا', 'الصين', 'الولايات المتحدة'],
            ['ما عاصمة أستراليا؟', 'كانبرا', 'سيدني', 'ملبورن', 'بيرث'],
            ['ما أعلى قمة جبلية في العالم؟', 'إفرست', 'كي 2', 'كانتشينجونجا', 'مون بلان'],
            ['ما أكبر محيط في العالم؟', 'الهادئ', 'الأطلسي', 'الهندي', 'المتجمد الشمالي'],
            ['ما عاصمة البرازيل؟', 'برازيليا', 'ريو دي جانيرو', 'ساو باولو', 'سلفادور'],
            ['ما أكبر صحراء حارة في العالم؟', 'الصحراء الكبرى', 'الربع الخالي', 'كالاهاري', 'غوبي'],
            ['ما عاصمة كندا؟', 'أوتاوا', 'تورونتو', 'مونتريال', 'فانكوفر'],
            ['ما أصغر دولة في العالم مساحة؟', 'الفاتيكان', 'موناكو', 'سان مارينو', 'ليختنشتاين'],
            ['ما عاصمة تركيا؟', 'أنقرة', 'إسطنبول', 'إزمير', 'أنطاليا'],
            ['أي بحر يفصل أوروبا عن أفريقيا؟', 'البحر المتوسط', 'البحر الأحمر', 'البحر الأسود', 'بحر قزوين'],
            ['ما عاصمة اليابان؟', 'طوكيو', 'كيوتو', 'أوساكا', 'ناغويا'],
            ['أي دولة عدد سكانها الأكبر؟', 'الهند', 'الصين', 'الولايات المتحدة', 'إندونيسيا'],
          ];

          class GeoApp extends StatelessWidget {
            const GeoApp({super.key});
            @override
            Widget build(BuildContext context) => MaterialApp(
                  debugShowCheckedModeBanner: false,
                  theme: ThemeData(useMaterial3: true, colorSchemeSeed: Colors.teal),
                  builder: (c, w) => Directionality(textDirection: TextDirection.rtl, child: w!),
                  home: const Quiz(),
                );
          }

          class Quiz extends StatefulWidget {
            const Quiz({super.key});
            @override
            State<Quiz> createState() => _QuizState();
          }

          class _QuizState extends State<Quiz> {
            late List<List<String>> qs;
            late List<String> opts;
            int i = 0, score = 0;
            bool done = false;
            String? picked;
            BannerAd? ad;

            @override
            void initState() {
              super.initState();
              start();
              ad = BannerAd(
                adUnitId: bannerId,
                size: AdSize.banner,
                request: const AdRequest(),
                listener: BannerAdListener(
                  onAdLoaded: (_) => setState(() {}),
                  onAdFailedToLoad: (a, e) {
                    a.dispose();
                    ad = null;
                  },
                ),
              )..load();
            }

            @override
            void dispose() {
              ad?.dispose();
              super.dispose();
            }

            void start() {
              qs = [...data]..shuffle();
              i = 0;
              score = 0;
              done = false;
              picked = null;
              load();
            }

            void load() {
              opts = qs[i].sublist(1)..shuffle();
            }

            void pick(String o) {
              if (picked != null) return;
              setState(() {
                picked = o;
                if (o == qs[i][1]) score++;
              });
            }

            void next() {
              setState(() {
                if (i < qs.length - 1) {
                  i++;
                  picked = null;
                  load();
                } else {
                  done = true;
                }
              });
            }

            @override
            Widget build(BuildContext context) {
              if (done) {
                return Scaffold(
                  body: Center(
                    child: Column(mainAxisSize: MainAxisSize.min, children: [
                      const Text('نتيجتك', style: TextStyle(fontSize: 28)),
                      Text('$score / ${qs.length}',
                          style: const TextStyle(fontSize: 56, fontWeight: FontWeight.bold)),
                      const SizedBox(height: 24),
                      FilledButton(
                          onPressed: () => setState(start),
                          child: const Text('العب مرة أخرى', style: TextStyle(fontSize: 18))),
                    ]),
                  ),
                );
              }
              return Scaffold(
                appBar: AppBar(title: const Text('اختبار الجغرافيا')),
                bottomNavigationBar: ad == null
                    ? null
                    : SizedBox(
                        height: ad!.size.height.toDouble(),
                        child: AdWidget(ad: ad!)),
                body: Padding(
                  padding: const EdgeInsets.all(20),
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.stretch,
                    children: [
                      LinearProgressIndicator(value: (i + 1) / qs.length),
                      const SizedBox(height: 8),
                      Text('السؤال ${i + 1} من ${qs.length}   |   النقاط: $score'),
                      const SizedBox(height: 24),
                      Text(qs[i][0],
                          style: const TextStyle(fontSize: 24, fontWeight: FontWeight.bold)),
                      const SizedBox(height: 24),
                      ...opts.map((o) {
                        Color? col;
                        if (picked != null) {
                          if (o == qs[i][1]) {
                            col = Colors.green;
                          } else if (o == picked) {
                            col = Colors.red;
                          }
                        }
                        return Padding(
                          padding: const EdgeInsets.only(bottom: 12),
                          child: FilledButton(
                            style: FilledButton.styleFrom(
                                backgroundColor: col, padding: const EdgeInsets.all(16)),
                            onPressed: () => pick(o),
                            child: Text(o, style: const TextStyle(fontSize: 18)),
                          ),
                        );
                      }),
                      const Spacer(),
                      if (picked != null)
                        FilledButton.tonal(
                            onPressed: next,
                            child: Text(i < qs.length - 1 ? 'التالي' : 'النتيجة')),
                    ],
                  ),
                ),
              );
            }
          }
          EOF
          flutter build apk --release
      - uses: actions/upload-artifact@v4
        with:
          name: geoquiz-apk
          path: build/app/outputs/flutter-apk/app-release.apk
