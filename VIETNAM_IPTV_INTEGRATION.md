# Hướng dẫn tích hợp kênh IPTV Việt Nam từ iptv-org

## Giới thiệu

Dự án iptv-org (https://github.com/iptv-org/iptv) là một trong những nguồn mở lớn nhất cung cấp danh sách kênh IPTV toàn cầu. Tài liệu này hướng dẫn chi tiết cách tích hợp các kênh IPTV Việt Nam từ nguồn này vào nền tảng OpenLunaAI.

## Nguồn dữ liệu

Danh sách kênh IPTV Việt Nam được lấy từ: https://github.com/iptv-org/iptv/blob/master/streams/vn.m3u

File này chứa các kênh truyền hình Việt Nam với định dạng M3U8 chuẩn, bao gồm thông tin như:
- Tên kênh
- Logo kênh
- Nhóm (thể loại)
- URL phát trực tiếp

## Triển khai tích hợp

### 1. Tạo API Endpoint để lấy và xử lý danh sách kênh

```typescript
// server/routes/vietnam-iptv-api.ts
import express from 'express';
import axios from 'axios';
import { parseM3U } from '../utils/m3u-parser';
import { Channel } from '@shared/schema';

const router = express.Router();

// Endpoint lấy danh sách kênh IPTV Việt Nam từ iptv-org
router.get('/vietnam', async (req, res) => {
  try {
    // URL của danh sách kênh IPTV Việt Nam từ iptv-org
    const iptvOrgVnUrl = 'https://iptv-org.github.io/iptv/countries/vn.m3u';
    
    // Tải danh sách kênh
    const response = await axios.get(iptvOrgVnUrl);
    const m3uContent = response.data;
    
    // Phân tích nội dung M3U thành danh sách kênh
    let channels = parseM3U(m3uContent);
    
    // Thêm trường nguồn để theo dõi
    channels = channels.map(channel => ({
      ...channel,
      source: 'iptv-org'
    }));
    
    // Lọc ra các kênh có URL hợp lệ
    channels = channels.filter(channel => {
      return channel.url && 
             (channel.url.startsWith('http://') || 
              channel.url.startsWith('https://'));
    });
    
    // Sắp xếp kênh theo nhóm và tên
    channels.sort((a, b) => {
      if (a.group !== b.group) {
        return a.group.localeCompare(b.group);
      }
      return a.name.localeCompare(b.name);
    });
    
    res.json(channels);
  } catch (error) {
    console.error('Error fetching Vietnam IPTV channels:', error);
    res.status(500).json({ 
      error: 'Failed to fetch Vietnam IPTV channels',
      message: error.message 
    });
  }
});

// Endpoint kiểm tra trạng thái kênh (có thể phát được không)
router.get('/vietnam/check/:channelId', async (req, res) => {
  try {
    const { channelId } = req.params;
    
    // Lấy danh sách kênh
    const iptvOrgVnUrl = 'https://iptv-org.github.io/iptv/countries/vn.m3u';
    const response = await axios.get(iptvOrgVnUrl);
    const m3uContent = response.data;
    const channels = parseM3U(m3uContent);
    
    // Tìm kênh cần kiểm tra
    const channel = channels.find(ch => ch.id === channelId);
    
    if (!channel) {
      return res.status(404).json({ error: 'Channel not found' });
    }
    
    // Kiểm tra kênh có thể phát được không
    try {
      // Gửi request HEAD để kiểm tra URL mà không tải toàn bộ nội dung
      await axios.head(channel.url, { timeout: 5000 });
      res.json({ id: channelId, status: 'online' });
    } catch (streamError) {
      res.json({ id: channelId, status: 'offline', error: streamError.message });
    }
  } catch (error) {
    console.error('Error checking channel status:', error);
    res.status(500).json({ error: 'Failed to check channel status' });
  }
});

export default router;
```

### 2. Xây dựng trang hiển thị kênh IPTV Việt Nam

```typescript
// client/src/pages/VietnamIPTVPage.tsx
import React, { useState, useEffect } from 'react';
import { useTranslation } from 'react-i18next';
import { 
  Tabs, 
  TabsContent, 
  TabsList, 
  TabsTrigger 
} from '@/components/ui/tabs';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { ScrollArea } from '@/components/ui/scroll-area';
import { Badge } from '@/components/ui/badge';
import { Search, RefreshCw } from 'lucide-react';
import IPTVPlayer from '@/components/IPTVPlayer';
import { Channel } from '@shared/schema';

// Định nghĩa các nhóm kênh phổ biến
const popularGroups = [
  'all',
  'general', 
  'news', 
  'entertainment', 
  'sports', 
  'education',
  'kids',
  'religious',
  'shop',
  'music',
  'local'
];

export default function VietnamIPTVPage() {
  const { t } = useTranslation();
  const [channels, setChannels] = useState<Channel[]>([]);
  const [filteredChannels, setFilteredChannels] = useState<Channel[]>([]);
  const [selectedChannel, setSelectedChannel] = useState<Channel | null>(null);
  const [activeGroup, setActiveGroup] = useState('all');
  const [searchTerm, setSearchTerm] = useState('');
  const [isLoading, setIsLoading] = useState(true);
  const [isRefreshing, setIsRefreshing] = useState(false);
  
  // Lấy danh sách kênh khi component được tải
  useEffect(() => {
    fetchChannels();
  }, []);
  
  // Lọc kênh khi nhóm hoặc từ khóa tìm kiếm thay đổi
  useEffect(() => {
    filterChannels();
  }, [channels, activeGroup, searchTerm]);
  
  // Hàm lấy danh sách kênh từ API
  const fetchChannels = async () => {
    try {
      setIsLoading(true);
      const response = await fetch('/api/iptv/vietnam');
      const data = await response.json();
      setChannels(data);
      
      // Mặc định chọn kênh đầu tiên nếu có
      if (data.length > 0 && !selectedChannel) {
        setSelectedChannel(data[0]);
      }
    } catch (error) {
      console.error('Failed to fetch Vietnam IPTV channels:', error);
    } finally {
      setIsLoading(false);
      setIsRefreshing(false);
    }
  };
  
  // Hàm làm mới danh sách kênh
  const refreshChannels = () => {
    setIsRefreshing(true);
    fetchChannels();
  };
  
  // Hàm lọc kênh theo nhóm và từ khóa tìm kiếm
  const filterChannels = () => {
    let filtered = [...channels];
    
    // Lọc theo nhóm
    if (activeGroup !== 'all') {
      filtered = filtered.filter(channel => 
        channel.group.toLowerCase() === activeGroup.toLowerCase()
      );
    }
    
    // Lọc theo từ khóa tìm kiếm
    if (searchTerm.trim()) {
      const term = searchTerm.toLowerCase();
      filtered = filtered.filter(channel => 
        channel.name.toLowerCase().includes(term) ||
        (channel.description && channel.description.toLowerCase().includes(term))
      );
    }
    
    setFilteredChannels(filtered);
  };
  
  // Nhóm kênh duy nhất từ danh sách kênh
  const getUniqueGroups = () => {
    const groups = new Set<string>();
    channels.forEach(channel => {
      groups.add(channel.group.toLowerCase());
    });
    return ['all', ...Array.from(groups)];
  };
  
  // Hiển thị tên nhóm đẹp hơn
  const formatGroupName = (group: string) => {
    if (group === 'all') return t('iptv.allChannels');
    return group.charAt(0).toUpperCase() + group.slice(1);
  };
  
  return (
    <div className="container mx-auto px-4 py-6">
      <h1 className="text-2xl font-bold mb-6">
        {t('iptv.vietnamChannels')}
      </h1>
      
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* Trình phát video kênh đang chọn */}
        <div className="lg:col-span-2">
          {selectedChannel ? (
            <div className="rounded-lg overflow-hidden bg-black aspect-video">
              <IPTVPlayer
                channelUrl={selectedChannel.url}
                channelName={selectedChannel.name}
                onError={() => console.log(`Error playing channel: ${selectedChannel.name}`)}
              />
            </div>
          ) : (
            <div className="rounded-lg overflow-hidden bg-gray-900 aspect-video flex items-center justify-center text-white">
              {isLoading ? (
                <p>{t('iptv.loading')}</p>
              ) : (
                <p>{t('iptv.selectChannel')}</p>
              )}
            </div>
          )}
          
          {/* Thông tin kênh đang chọn */}
          {selectedChannel && (
            <div className="mt-4 p-4 bg-gray-100 dark:bg-gray-800 rounded-lg">
              <div className="flex items-center">
                {selectedChannel.logo && (
                  <img 
                    src={selectedChannel.logo} 
                    alt={selectedChannel.name}
                    className="w-12 h-12 mr-4 object-contain"
                    onError={(e) => {
                      (e.target as HTMLImageElement).style.display = 'none';
                    }}
                  />
                )}
                <div>
                  <h2 className="text-xl font-semibold">{selectedChannel.name}</h2>
                  <div className="flex items-center mt-1">
                    <Badge className="mr-2">{formatGroupName(selectedChannel.group)}</Badge>
                    <span className="text-sm text-gray-500">ID: {selectedChannel.id}</span>
                  </div>
                </div>
              </div>
              {selectedChannel.description && (
                <p className="mt-3 text-sm">{selectedChannel.description}</p>
              )}
            </div>
          )}
        </div>
        
        {/* Danh sách kênh */}
        <div className="bg-white dark:bg-gray-800 rounded-lg shadow p-4">
          {/* Thanh tìm kiếm */}
          <div className="relative mb-4">
            <Input
              type="text"
              placeholder={t('iptv.searchChannels')}
              value={searchTerm}
              onChange={(e) => setSearchTerm(e.target.value)}
              className="pl-10"
            />
            <Search className="absolute left-3 top-1/2 transform -translate-y-1/2 h-4 w-4 text-gray-400" />
            <Button 
              variant="ghost" 
              size="icon" 
              className="absolute right-2 top-1/2 transform -translate-y-1/2"
              onClick={refreshChannels}
              disabled={isRefreshing}
            >
              <RefreshCw className={`h-4 w-4 ${isRefreshing ? 'animate-spin' : ''}`} />
            </Button>
          </div>
          
          {/* Tabs cho các nhóm kênh */}
          <Tabs defaultValue="all" value={activeGroup} onValueChange={setActiveGroup}>
            <TabsList className="mb-4 flex flex-wrap">
              {popularGroups.map(group => (
                <TabsTrigger key={group} value={group} className="text-xs">
                  {formatGroupName(group)}
                </TabsTrigger>
              ))}
            </TabsList>
            
            <TabsContent value={activeGroup} className="mt-0">
              {isLoading ? (
                <div className="flex justify-center items-center h-64">
                  <p>{t('iptv.loading')}</p>
                </div>
              ) : filteredChannels.length === 0 ? (
                <div className="text-center py-10">
                  <p>{t('iptv.noChannels')}</p>
                </div>
              ) : (
                <ScrollArea className="h-[500px] pr-4">
                  <div className="space-y-2">
                    {filteredChannels.map(channel => (
                      <div
                        key={channel.id}
                        className={`flex items-center p-3 rounded-md cursor-pointer hover:bg-gray-100 dark:hover:bg-gray-700 transition-colors ${
                          selectedChannel?.id === channel.id ? 'bg-blue-50 dark:bg-gray-700' : ''
                        }`}
                        onClick={() => setSelectedChannel(channel)}
                      >
                        {channel.logo ? (
                          <img
                            src={channel.logo}
                            alt={channel.name}
                            className="w-8 h-8 mr-3 object-contain"
                            onError={(e) => {
                              (e.target as HTMLImageElement).src = '/images/tv-placeholder.png';
                            }}
                          />
                        ) : (
                          <div className="w-8 h-8 mr-3 bg-gray-200 dark:bg-gray-700 rounded-full flex items-center justify-center">
                            <span className="text-xs font-bold">
                              {channel.name.charAt(0)}
                            </span>
                          </div>
                        )}
                        <div className="flex-1 min-w-0">
                          <h3 className="font-medium text-sm truncate">{channel.name}</h3>
                          <p className="text-xs text-gray-500 dark:text-gray-400">
                            {formatGroupName(channel.group)}
                          </p>
                        </div>
                      </div>
                    ))}
                  </div>
                </ScrollArea>
              )}
            </TabsContent>
          </Tabs>
        </div>
      </div>
    </div>
  );
}
```

### 3. Bổ sung tiện ích lưu kênh yêu thích

```typescript
// client/src/components/FavoriteChannelButton.tsx
import React from 'react';
import { Star } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { useTranslation } from 'react-i18next';
import { useFavoriteChannels } from '@/hooks/use-favorite-channels';
import { Channel } from '@shared/schema';

interface FavoriteChannelButtonProps {
  channel: Channel;
}

const FavoriteChannelButton: React.FC<FavoriteChannelButtonProps> = ({ channel }) => {
  const { t } = useTranslation();
  const { isFavorite, toggleFavorite } = useFavoriteChannels();
  
  const isChannelFavorite = isFavorite(channel.id);
  
  return (
    <Button
      variant="ghost"
      size="sm"
      onClick={(e) => {
        e.stopPropagation();
        toggleFavorite(channel);
      }}
      className={`p-1 h-8 w-8 ${isChannelFavorite ? 'text-yellow-500' : 'text-gray-400'}`}
      title={isChannelFavorite ? t('iptv.removeFromFavorites') : t('iptv.addToFavorites')}
    >
      <Star className="h-5 w-5" fill={isChannelFavorite ? 'currentColor' : 'none'} />
    </Button>
  );
};

export default FavoriteChannelButton;
```

### 4. Hook quản lý kênh yêu thích

```typescript
// client/src/hooks/use-favorite-channels.ts
import { useState, useEffect } from 'react';
import { Channel } from '@shared/schema';

export function useFavoriteChannels() {
  const [favorites, setFavorites] = useState<Channel[]>([]);
  
  // Khởi tạo từ localStorage khi component được tải
  useEffect(() => {
    const storedFavorites = localStorage.getItem('favoriteChannels');
    if (storedFavorites) {
      try {
        setFavorites(JSON.parse(storedFavorites));
      } catch (error) {
        console.error('Error parsing favorite channels', error);
        localStorage.removeItem('favoriteChannels');
      }
    }
  }, []);
  
  // Kiểm tra xem một kênh có nằm trong danh sách yêu thích không
  const isFavorite = (channelId: string): boolean => {
    return favorites.some(fav => fav.id === channelId);
  };
  
  // Thêm/xóa kênh khỏi danh sách yêu thích
  const toggleFavorite = (channel: Channel) => {
    let updatedFavorites: Channel[];
    
    if (isFavorite(channel.id)) {
      // Xóa khỏi danh sách yêu thích
      updatedFavorites = favorites.filter(fav => fav.id !== channel.id);
    } else {
      // Thêm vào danh sách yêu thích
      updatedFavorites = [...favorites, channel];
    }
    
    // Cập nhật state và localStorage
    setFavorites(updatedFavorites);
    localStorage.setItem('favoriteChannels', JSON.stringify(updatedFavorites));
  };
  
  // Lấy danh sách kênh yêu thích
  const getFavoriteChannels = (): Channel[] => {
    return favorites;
  };
  
  return {
    favorites: getFavoriteChannels(),
    isFavorite,
    toggleFavorite
  };
}
```

### 5. Cập nhật trình phát IPTV để hỗ trợ nguồn phát dự phòng

```typescript
// client/src/components/IPTVPlayerEnhanced.tsx
import React, { useState, useEffect, useRef } from 'react';
import Hls from 'hls.js';
import { useTranslation } from 'react-i18next';

interface IPTVPlayerProps {
  channelUrl: string;
  channelName: string;
  fallbackUrls?: string[]; // URLs dự phòng nếu URL chính không hoạt động
  onError?: () => void;
}

const IPTVPlayerEnhanced: React.FC<IPTVPlayerProps> = ({
  channelUrl,
  channelName,
  fallbackUrls = [],
  onError
}) => {
  const { t } = useTranslation();
  const videoRef = useRef<HTMLVideoElement>(null);
  const [currentUrlIndex, setCurrentUrlIndex] = useState(0);
  const [errorCount, setErrorCount] = useState(0);
  const [hasError, setHasError] = useState(false);
  const [isIframe, setIsIframe] = useState(false);
  
  // Tổng hợp tất cả các URL có thể
  const allUrls = [channelUrl, ...fallbackUrls];
  const currentUrl = allUrls[currentUrlIndex];
  
  // Xác định loại URL để sử dụng phương pháp phát thích hợp
  useEffect(() => {
    if (currentUrl.includes('embed') || 
        !currentUrl.endsWith('.m3u8') || 
        currentUrl.includes('youtube') ||
        currentUrl.includes('facebook')) {
      setIsIframe(true);
    } else {
      setIsIframe(false);
      setupHlsPlayer();
    }
  }, [currentUrl]);
  
  // Thiết lập trình phát HLS
  const setupHlsPlayer = () => {
    if (isIframe) return;
    
    const video = videoRef.current;
    if (!video) return;
    
    // Xóa instance HLS cũ nếu có
    if (video.hls) {
      video.hls.destroy();
      delete video.hls;
    }
    
    // Kiểm tra nếu trình duyệt hỗ trợ HLS nội bộ
    if (video.canPlayType('application/vnd.apple.mpegurl')) {
      // Trình duyệt Safari hỗ trợ HLS nội bộ
      video.src = currentUrl;
    } else if (Hls.isSupported()) {
      // Sử dụng thư viện HLS.js
      const hls = new Hls({
        maxBufferLength: 30,
        maxMaxBufferLength: 60,
        manifestLoadingTimeOut: 10000,
        manifestLoadingMaxRetry: 5,
        levelLoadingTimeOut: 10000,
        levelLoadingMaxRetry: 5
      });
      
      // Xử lý lỗi từ HLS.js
      hls.on(Hls.Events.ERROR, (_, data) => {
        if (data.fatal) {
          console.error('Fatal HLS error:', data);
          setErrorCount(prev => prev + 1);
          
          if (errorCount >= 3) {
            // Thử URL tiếp theo sau 3 lỗi nghiêm trọng
            tryNextUrl();
          } else {
            // Thử tải lại
            hls.destroy();
            setupHlsPlayer();
          }
        }
      });
      
      hls.attachMedia(video);
      hls.loadSource(currentUrl);
      hls.on(Hls.Events.MANIFEST_PARSED, () => {
        video.play().catch(error => {
          console.warn('Autoplay failed:', error);
        });
      });
      
      // Lưu instance HLS vào video element để có thể dọn dẹp sau này
      video.hls = hls;
    } else {
      // Trình duyệt không hỗ trợ HLS
      console.warn('Browser does not support HLS');
      setHasError(true);
    }
  };
  
  // Thử URL tiếp theo khi có lỗi
  const tryNextUrl = () => {
    if (currentUrlIndex < allUrls.length - 1) {
      setCurrentUrlIndex(prev => prev + 1);
      setErrorCount(0);
    } else {
      setHasError(true);
      if (onError) onError();
    }
  };
  
  // Xử lý lỗi phát video
  const handleVideoError = () => {
    console.error('Video playback error');
    setErrorCount(prev => prev + 1);
    
    if (errorCount >= 2) {
      tryNextUrl();
    }
  };
  
  // Dọn dẹp khi unmount
  useEffect(() => {
    return () => {
      const video = videoRef.current;
      if (video && video.hls) {
        video.hls.destroy();
        delete video.hls;
      }
    };
  }, []);
  
  // Hiển thị thông báo lỗi khi không thể phát
  if (hasError) {
    return (
      <div className="flex flex-col items-center justify-center h-full bg-gray-900 text-white p-4">
        <p className="mb-2 text-lg font-semibold">{t('iptv.playbackError')}</p>
        <p className="mb-4 text-center">{t('iptv.unableToPlay', { name: channelName })}</p>
        <a 
          href={channelUrl} 
          target="_blank" 
          rel="noopener noreferrer"
          className="px-4 py-2 bg-blue-600 rounded hover:bg-blue-700 transition-colors"
        >
          {t('iptv.openInBrowser')}
        </a>
      </div>
    );
  }
  
  // Sử dụng iframe cho các URL không phải HLS
  if (isIframe) {
    return (
      <div className="relative w-full h-full">
        <iframe
          src={currentUrl}
          className="absolute inset-0 w-full h-full border-0"
          allowFullScreen
          title={channelName}
          sandbox="allow-same-origin allow-scripts allow-forms"
          referrerPolicy="no-referrer-when-downgrade"
        ></iframe>
      </div>
    );
  }
  
  // Sử dụng video player cho các URL HLS
  return (
    <div className="relative w-full h-full">
      <video
        ref={videoRef}
        className="w-full h-full"
        controls
        autoPlay
        playsInline
        onError={handleVideoError}
      ></video>
    </div>
  );
};

export default IPTVPlayerEnhanced;
```

### 6. Thêm vào đường dẫn chính

```typescript
// server/routes.ts
import vietnamIptvApiRouter from './routes/vietnam-iptv-api';

// ...trong phần app.use
app.use('/api/iptv', vietnamIptvApiRouter);
```

```typescript
// client/src/App.tsx
import VietnamIPTVPage from '@/pages/VietnamIPTVPage';

// ...trong phần Route
<Route path="/iptv/vietnam" component={VietnamIPTVPage} />
```

### 7. Thêm liên kết vào menu điều hướng

```typescript
// client/src/components/Navbar.tsx
<DropdownMenuItem asChild>
  <Link to="/iptv/vietnam">
    {t('navigation.vietnamTV')}
  </Link>
</DropdownMenuItem>
```

## Cái tiến bổ sung

### 1. Thu thập lịch phát sóng (EPG)

Lịch phát sóng giúp hiển thị thông tin chương trình đang phát và sắp phát:

```typescript
// server/routes/vietnam-epg-api.ts
import express from 'express';
import axios from 'axios';
import { parseStringPromise } from 'xml2js';

const router = express.Router();

router.get('/vietnam', async (req, res) => {
  try {
    // URL của EPG Việt Nam, ví dụ từ iptv-org
    const epgUrl = 'https://iptv-org.github.io/epg/guides/vn/vtv.vn.xml';
    
    const response = await axios.get(epgUrl);
    const xmlData = response.data;
    
    // Parse dữ liệu XML
    const result = await parseStringPromise(xmlData, {
      explicitArray: false,
      mergeAttrs: true
    });
    
    // Chuyển đổi sang định dạng dễ sử dụng
    const programmes = result.tv.programme;
    const formattedEpg = Array.isArray(programmes) 
      ? programmes.map(formatProgramme) 
      : [formatProgramme(programmes)];
    
    res.json(formattedEpg);
  } catch (error) {
    console.error('Error fetching Vietnam EPG:', error);
    res.status(500).json({ error: 'Failed to fetch EPG data' });
  }
});

// Hàm định dạng dữ liệu chương trình
function formatProgramme(programme) {
  return {
    channelId: programme.channel,
    start: programme.start,
    stop: programme.stop,
    title: programme.title?._,
    description: programme.desc?._,
    category: programme.category?._,
    language: programme.title?.$.lang || 'vi'
  };
}

export default router;
```

### 2. Quản lý lịch sử xem và đề xuất

```typescript
// client/src/hooks/use-viewing-history.ts
import { useState, useEffect } from 'react';
import { Channel } from '@shared/schema';

interface ViewingRecord {
  channelId: string;
  channelName: string;
  lastViewed: number; // timestamp
  viewCount: number;
}

export function useViewingHistory() {
  const [history, setHistory] = useState<ViewingRecord[]>([]);
  
  // Khởi tạo từ localStorage
  useEffect(() => {
    const storedHistory = localStorage.getItem('viewingHistory');
    if (storedHistory) {
      try {
        setHistory(JSON.parse(storedHistory));
      } catch (error) {
        console.error('Error parsing viewing history', error);
        localStorage.removeItem('viewingHistory');
      }
    }
  }, []);
  
  // Cập nhật lịch sử khi xem một kênh
  const recordViewing = (channel: Channel) => {
    const now = Date.now();
    let updatedHistory = [...history];
    
    const existingIndex = updatedHistory.findIndex(item => item.channelId === channel.id);
    
    if (existingIndex >= 0) {
      // Cập nhật bản ghi hiện có
      updatedHistory[existingIndex] = {
        ...updatedHistory[existingIndex],
        lastViewed: now,
        viewCount: updatedHistory[existingIndex].viewCount + 1
      };
    } else {
      // Tạo bản ghi mới
      updatedHistory.push({
        channelId: channel.id,
        channelName: channel.name,
        lastViewed: now,
        viewCount: 1
      });
    }
    
    // Giới hạn lịch sử đến 50 kênh
    if (updatedHistory.length > 50) {
      updatedHistory.sort((a, b) => b.lastViewed - a.lastViewed);
      updatedHistory = updatedHistory.slice(0, 50);
    }
    
    // Cập nhật state và localStorage
    setHistory(updatedHistory);
    localStorage.setItem('viewingHistory', JSON.stringify(updatedHistory));
  };
  
  // Lấy lịch sử xem gần đây
  const getRecentlyViewed = (limit = 10): ViewingRecord[] => {
    return [...history]
      .sort((a, b) => b.lastViewed - a.lastViewed)
      .slice(0, limit);
  };
  
  // Lấy kênh được xem nhiều nhất
  const getMostViewed = (limit = 10): ViewingRecord[] => {
    return [...history]
      .sort((a, b) => b.viewCount - a.viewCount)
      .slice(0, limit);
  };
  
  return {
    recordViewing,
    getRecentlyViewed,
    getMostViewed
  };
}
```

## Tích hợp với hệ thống VPN

Để truy cập các kênh Việt Nam từ nước ngoài, tích hợp với hệ thống VPN đã có:

```typescript
// client/src/components/VietnamIPTVVpnOverlay.tsx
import React from 'react';
import { Shield } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { useTranslation } from 'react-i18next';

interface VietnamIPTVVpnOverlayProps {
  onEnableVpn: () => void;
  isEnabling: boolean;
}

const VietnamIPTVVpnOverlay: React.FC<VietnamIPTVVpnOverlayProps> = ({
  onEnableVpn,
  isEnabling
}) => {
  const { t } = useTranslation();
  
  return (
    <div className="absolute inset-0 bg-black bg-opacity-80 flex flex-col items-center justify-center text-white p-6 text-center">
      <Shield className="h-16 w-16 mb-4 text-orange-500" />
      <h3 className="text-xl font-bold mb-2">
        {t('iptv.vietnamRegionRestricted')}
      </h3>
      <p className="mb-6 max-w-md">
        {t('iptv.vietnamVpnRequired')}
      </p>
      <Button
        onClick={onEnableVpn}
        disabled={isEnabling}
        className="bg-orange-600 hover:bg-orange-700"
        size="lg"
      >
        {isEnabling ? t('iptv.connecting') : t('iptv.enableVietnamVpn')}
        {isEnabling && (
          <span className="ml-2 animate-spin">⟳</span>
        )}
      </Button>
    </div>
  );
};

export default VietnamIPTVVpnOverlay;
```

## Lưu ý quan trọng

1. **Kiểm tra tính khả dụng của kênh**:
   - Nhiều kênh từ iptv-org có thể không hoạt động hoặc chất lượng không ổn định
   - Triển khai kiểm tra tính khả dụng định kỳ và cập nhật trạng thái kênh

2. **Vấn đề pháp lý**:
   - Cần đảm bảo việc sử dụng các nguồn IPTV này là hợp pháp tại khu vực của bạn
   - Sử dụng nội dung với mục đích cá nhân và giáo dục

3. **Vấn đề hiệu suất**:
   - Cân nhắc sử dụng lưu trữ đệm (caching) để giảm tải cho server
   - Lưu danh sách kênh trong cơ sở dữ liệu và cập nhật định kỳ thay vì tải mỗi lần truy cập

4. **Cải thiện trải nghiệm người dùng**:
   - Cung cấp thông tin về chất lượng kênh (SD, HD, FHD)
   - Thêm đánh giá và phản hồi từ người dùng
   - Lưu các cài đặt như âm lượng, kênh gần đây, v.v.

5. **Cập nhật danh sách kênh**:
   - Thiết lập công việc định kỳ để cập nhật danh sách kênh từ iptv-org
   - Kết hợp các nguồn IPTV khác để tăng tính sẵn sàng