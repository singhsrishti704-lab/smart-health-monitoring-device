clc; clear; close all;

%% ================= USER SETTINGS =================
portName = "/dev/cu.usbmodem101";   % change if needed
baudRate = 115200;
duration = 10;                      % seconds
baselineSeconds = 5;
ALARM_BPM = 80;                     % buzzer if BPM > 80
% ==================================================

%% ================= SERIAL SETUP =================
s = serialport(portName, baudRate);
configureTerminator(s,"LF");
flush(s);

fprintf("\n\033[1;34m============================================\033[0m\n");
fprintf("\033[1;34mConnected to %s @ %d baud\033[0m\n", portName, baudRate);
fprintf("\033[1;34mCollecting data for %d seconds...\033[0m\n", duration);
fprintf("\033[1;34m============================================\033[0m\n\n");

%% ================= DATA COLLECTION =================
t_raw=[]; ir_raw=[]; red_raw=[]; temp_raw=[]; rf_raw=[];
t0 = tic;
lastPrint = -1;

while toc(t0) < duration
    elapsed = floor(toc(t0));
    if elapsed ~= lastPrint
        fprintf("\033[1;33mCollecting data... %d / %d seconds\033[0m\n", elapsed, duration);
        lastPrint = elapsed;
    end

    if s.NumBytesAvailable == 0
        pause(0.01); continue;
    end

    line = strtrim(readline(s));
    if line=="" || contains(line,"ERROR") || contains(line,"START")
        continue;
    end

    v = str2double(split(line,","));
    if numel(v)>=5 && all(~isnan(v))
        t_raw(end+1)=v(1);
        ir_raw(end+1)=v(2);
        red_raw(end+1)=v(3);
        temp_raw(end+1)=v(4);
        rf_raw(end+1)=v(5);
    end
end

fprintf("\n\033[1;32mData collection completed.\033[0m\n");
fprintf("\033[1;32mTotal samples collected: %d\033[0m\n\n", numel(t_raw));

%% ================= PREPROCESS =================
t = (t_raw - t_raw(1))/1000;
[t,idx] = unique(t,'stable');

ir = ir_raw(idx);
red = red_raw(idx);
temp = temp_raw(idx);

fs = numel(t)/t(end);

%% ================= HEART RATE =================
ir = ir - mean(ir);
ir = movmedian(ir, round(0.02*fs));
ir = movmean(ir, round(0.05*fs));

irN = (ir - min(ir)) / (max(ir)-min(ir)+eps);

[pks,locs] = findpeaks(irN, ...
    'MinPeakDistance', round(0.45*fs), ...
    'MinPeakHeight', 0.4);

if numel(locs) >= 2
    RR = diff(t(locs));
    HR = mean(60 ./ RR);
else
    HR = NaN;
end

%% ================= TEMPERATURE =================
avgTemp = mean(temp);

%% ================= HEMOGLOBIN =================
baseIdx = t <= baselineSeconds;
ratio = mean(red) / mean(red(baseIdx));
hemoScore = max(0,min(100,(ratio-0.7)/(1.3-0.7)*100));

if hemoScore < 35
    hemoText = "Low";
elseif hemoScore < 55
    hemoText = "Borderline";
else
    hemoText = "Normal";
end

%% ================= BUZZER ALERT =================
if ~isnan(HR) && HR > ALARM_BPM
    fprintf("\033[1;31mALERT: Heart Rate > %d bpm. Buzzer ON!\033[0m\n", ALARM_BPM);
    try
        writeline(s,"ALARM");
        pause(2);
        writeline(s,"STOP");
    catch
        warning("Buzzer communication failed.");
    end
end

%% ================= COMMAND WINDOW SUMMARY =================
fprintf("\n\n");
fprintf("\033[1;36m====================================================\033[0m\n");
fprintf("\033[1;36m                HEALTH SUMMARY                      \033[0m\n");
fprintf("\033[1;36m====================================================\033[0m\n\n");

if isnan(HR)
    fprintf("   \033[1;31mHeart Rate      :  Not detected\033[0m\n\n");
else
    fprintf("   \033[1;31mHeart Rate      :  %.1f bpm\033[0m\n\n", HR);
end

fprintf("   \033[1;32mTemperature     :  %.2f °C\033[0m\n\n", avgTemp);
fprintf("   \033[1;35mHemoglobin      :  %.0f (%s)\033[0m\n\n", hemoScore, hemoText);

fprintf("\033[1;36m====================================================\033[0m\n\n");

%% ================= DASHBOARD (NO OVERLAP) =================
figure('Name','Health Dashboard','Color','w','Position',[80 80 1200 760]);

% ---- TITLE ----
annotation('textbox',[0 0.92 1 0.06],'String','HEALTH DASHBOARD', ...
    'FontSize',26,'FontWeight','bold','EdgeColor','none', ...
    'HorizontalAlignment','center');

% ---- SUMMARY TEXT (TOP AREA ONLY) ----
annotation('textbox',[0.05 0.82 0.25 0.08], ...
    'String',sprintf('Heart Rate\n%.1f bpm',HR), ...
    'FontSize',24,'FontWeight','bold','Color',[0.8 0 0],'EdgeColor','none');

annotation('textbox',[0.37 0.82 0.25 0.08], ...
    'String',sprintf('Temperature\n%.2f °C',avgTemp), ...
    'FontSize',24,'FontWeight','bold','Color',[0 0.6 0],'EdgeColor','none');

annotation('textbox',[0.69 0.82 0.25 0.08], ...
    'String',sprintf('Hemoglobin\n%.0f',hemoScore), ...
    'FontSize',24,'FontWeight','bold','Color',[0.5 0 0.7],'EdgeColor','none');

% ---- PLOTS (MOVED DOWN – NO MIXING) ----
subplot('Position',[0.08 0.52 0.84 0.20])
plot(t, irN, 'r','LineWidth',2); hold on
plot(t(locs), irN(locs), 'yo','MarkerFaceColor','y')
title('PPG (IR) - Smoothed','FontWeight','bold')
ylabel('Amplitude'); grid on

subplot('Position',[0.08 0.28 0.84 0.18])
plot(t, temp,'g','LineWidth',2)
yline(37,'--r','LineWidth',1.5)
title('Skin Temperature (LM35)','FontWeight','bold')
ylabel('°C'); grid on

subplot('Position',[0.08 0.06 0.84 0.16])
plot(t, red,'b','LineWidth',2); hold on
yline(mean(red(baseIdx)),':k','Baseline','LineWidth',1.5)
title('TCS3200 Red Signal','FontWeight','bold')
xlabel('Time (s)'); grid on
